-- Tải thư viện Rayfield UI
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- 1. Khởi tạo cửa sổ chính
local Window = Rayfield:CreateWindow({
   Name = "📐 Xây 1 kim tự tháp | AI HACK",
   Icon = 0,
   LoadingTitle = "Đang load...",
   LoadingSubtitle = "by alatfera script",
   Theme = "AmberGlow",

   DisableRayfieldPrompts = false,
   DisableBuildWarnings = false,

   ConfigurationSaving = {
      Enabled = true,
      FolderName = "PyramidHubConfig",
      FileName = "Settings"
   },

   KeySystem = false
})

-- 2. Khai báo Services & Variables
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local playerGui = LocalPlayer and LocalPlayer:WaitForChild("PlayerGui", 10)

-- Remote references
local KnitServices = ReplicatedStorage:WaitForChild("Packages"):WaitForChild("_Index"):WaitForChild("sleitnick_knit@1.7.0"):WaitForChild("knit"):WaitForChild("Services")
local DataServiceRF = KnitServices:WaitForChild("DataService"):WaitForChild("RF"):WaitForChild("PurchaseUpgrade")
local BookServiceRF = KnitServices:WaitForChild("BookService"):WaitForChild("RF"):WaitForChild("Pickup")
local GymServiceRF = KnitServices:WaitForChild("GymService"):WaitForChild("RF"):WaitForChild("StartBench")

-- Trạng thái Remote Đặt khối
local PlaceServiceRF = KnitServices:FindFirstChild("PlacementService") 
    or KnitServices:FindFirstChild("BuildService") 
    or KnitServices:FindFirstChild("BlockService")

-- Trạng thái
local autoFarmActive = false
local speedTrainActive = false
local autoGymActive = false
local PICKUP_LOCATION = Vector3.new(-191.54, -5.88, 43.20)

-- 3. Các Tab
local FarmTab = Window:CreateTab("Farm", 4483362458)
local UpgradeTab = Window:CreateTab("Nâng cấp", 4483362458)

-- ==================== HÀM KIỂM TRA SỐ LƯỢNG KIM TỰ THÁP ====================

local function GetPyramidsAmount()
    if not LocalPlayer then return nil end
    local pGui = LocalPlayer:FindFirstChild("PlayerGui")
    if not pGui then return nil end

    -- 1. Tìm chính xác theo đường dẫn cấu trúc:
    local screenGui = pGui:FindFirstChild("ScreenGui")
    if screenGui then
        local centerLeft = screenGui:FindFirstChild("CenterLeft")
        if centerLeft then
            local leftSideBar = centerLeft:FindFirstChild("LeftSideBar")
            if leftSideBar then
                local pyramidsCounter = leftSideBar:FindFirstChild("PyramidsCounter")
                if pyramidsCounter then
                    local readout = pyramidsCounter:FindFirstChild("Readout")
                    if readout then
                        local amount = readout:FindFirstChild("Amount")
                        if amount then
                            local dropShadow = amount:FindFirstChild("DropShadow")
                            if dropShadow and (dropShadow:IsA("TextLabel") or dropShadow:IsA("TextBox")) then
                                return dropShadow.Text
                            end
                        end
                    end
                end
            end
        end
    end

    -- 2. Phương án dự phòng:
    for _, obj in ipairs(pGui:GetDescendants()) do
        if obj.Name == "PyramidsCounter" then
            local dropShadow = obj:FindFirstChild("DropShadow", true)
            if dropShadow and (dropShadow:IsA("TextLabel") or dropShadow:IsA("TextBox")) then
                return dropShadow.Text
            end
        end
    end

    return nil
end

-- ==================== HÀM KIỂM TRA STRENGTH / TÚI ====================

local function GetStrengthDetail()
    if not playerGui then return nil end

    local rawText = nil

    local sg = playerGui:FindFirstChild("ScreenGui")
    if sg then
        local centerLeft = sg:FindFirstChild("CenterLeft")
        if centerLeft then
            local leftSideBar = centerLeft:FindFirstChild("LeftSideBar")
            if leftSideBar then
                local strengthCounter = leftSideBar:FindFirstChild("StrengthCounter")
                if strengthCounter then
                    local readout = strengthCounter:FindFirstChild("Readout")
                    if readout then
                        local detail = readout:FindFirstChild("Detail")
                        if detail then
                            local dropShadow = detail:FindFirstChild("DropShadow") or detail:FindFirstChildOfClass("TextLabel")
                            if dropShadow then
                                rawText = dropShadow.Text
                            else
                                rawText = detail.Text
                            end
                        end
                    end
                end
            end
        end
    end

    if not rawText then
        for _, obj in ipairs(playerGui:GetDescendants()) do
            if obj.Name == "Detail" then
                local shadow = obj:FindFirstChild("DropShadow") or obj:FindFirstChildOfClass("TextLabel")
                if shadow and string.find(shadow.Text, "/") then
                    rawText = shadow.Text
                    break
                end
            end
        end
    end

    if rawText then
        local matched = string.match(rawText, "%d+/%d+")
        if matched then
            return matched
        end
        
        local cleaned = string.gsub(rawText, "[^%d/]", "")
        if cleaned ~= "" then
            return cleaned
        end
    end

    return nil
end

local function GetBagInfo()
    local str = GetStrengthDetail()
    if not str then return 0, 0 end
    
    local current, max = string.match(str, "(%d+)/(%d+)")
    if current and max then
        return tonumber(current) or 0, tonumber(max) or 0
    end
    return 0, 0
end

-- ==================== HÀM THỰC THI ĐẶT KHỐI ====================

local function TriggerPlaceBlock(targetBlock)
    if PlaceServiceRF then
        local rf = PlaceServiceRF:FindFirstChildOfClass("RemoteFunction") or PlaceServiceRF:FindFirstChild("RF")
        if rf then
            pcall(function()
                rf:InvokeServer(targetBlock)
            end)
        end
    end
end

-- ==================== LOGIC DỊCH CHUYỂN ====================

local function startAutoTeleport()
    local pyramidBuild = workspace:FindFirstChild("PyramidBuild")
    
    if not pyramidBuild then
        warn("Không tìm thấy folder 'PyramidBuild' trong Workspace!")
        return
    end

    local validBlocks = {}

    for _, layer in ipairs(pyramidBuild:GetChildren()) do
        local layerName = layer.Name
        if layerName:sub(1, 5) == "Layer" and not layerName:find("Slab") then
            for _, sparseGroup in ipairs(layer:GetChildren()) do
                if sparseGroup:IsA("Folder") then
                    for _, blockPart in ipairs(sparseGroup:GetChildren()) do
                        if blockPart:IsA("BasePart") then
                            table.insert(validBlocks, blockPart)
                        end
                    end
                elseif sparseGroup:IsA("BasePart") then
                    table.insert(validBlocks, sparseGroup)
                end
            end
        end
    end

    table.sort(validBlocks, function(a, b)
        return a.Position.Y < b.Position.Y
    end)

    local TELEPORT_DELAY = 0.7 

    for _, blockPart in ipairs(validBlocks) do
        if not autoFarmActive then break end

        local curCount, maxCount = GetBagInfo()
        if maxCount > 0 and curCount <= 0 then
            break 
        end

        if blockPart and blockPart.Parent then
            local character = LocalPlayer.Character
            local hrp = character and character:FindFirstChild("HumanoidRootPart")
            local camera = Workspace.CurrentCamera

            if hrp then
                local blockPos = blockPart.Position

                character:PivotTo(CFrame.new(blockPos + Vector3.new(0, 1.5, 0)))
                
                if camera then
                    camera.CFrame = CFrame.new(camera.CFrame.Position, blockPos)
                end
                
                task.wait(0.1)

                TriggerPlaceBlock(blockPart)

                task.wait(TELEPORT_DELAY)
            end
        end
    end
end

-- ==================== TAB FARM ====================

FarmTab:CreateButton({
   Name = "🔍 Kiểm tra số khối hiện có",
   Callback = function()
       local strength = GetStrengthDetail()
       if strength then
           Rayfield:Notify({
               Title = "Sức chứa hiện tại",
               Content = "Số khối: " .. strength,
               Duration = 4,
               Image = 4483362458,
           })
       else
           Rayfield:Notify({
               Title = "Thông báo",
               Content = "Không tìm thấy UI túi (Sẽ tự động farm bình thường)",
               Duration = 4,
               Image = 4483362458,
           })
       end
   end,
})

-- Toggle Auto Xây
FarmTab:CreateToggle({
   Name = "⚡ Auto xây kim tự tháp (Người chơi cần click)",
   CurrentValue = false,
   Flag = "AutoPyramidToggle",
   Callback = function(Value)
       autoFarmActive = Value
       
       if autoFarmActive then
           Rayfield:Notify({
               Title = "Auto Farm System",
               Content = "Đã bật Auto Xây Kim Tự Tháp (Tốc độ lụm: 0.4s)!",
               Duration = 4,
               Image = 4483362458,
           })
           
           task.spawn(function()
               local region = workspace:WaitForChild("Regions"):WaitForChild("Region:Quarry")
               
               while autoFarmActive do
                   local current, maxVal = GetBagInfo()
                   
                   -- 1. ĐI LỤM KHỐI
                   if current < maxVal or maxVal == 0 then
                       local char = LocalPlayer.Character
                       if char and char:FindFirstChild("HumanoidRootPart") then
                           char.HumanoidRootPart.CFrame = CFrame.new(PICKUP_LOCATION)
                       end
                       
                       local pickupStartTime = tick()
                       
                       while autoFarmActive do
                           local curCount, maxCount = GetBagInfo()
                           
                           if maxCount > 0 and curCount >= maxCount then
                               break 
                           end

                           if tick() - pickupStartTime > 8 then
                               break
                           end

                           pcall(function()
                               BookServiceRF:InvokeServer(region)
                           end)
                           
                           -- Đã tăng tốc độ lụm khối thành 0.4s
                           task.wait(0.4)
                       end
                   end
                   
                   -- 2. DỊCH CHUYỂN TỚI VỊ TRÍ XÂY
                   if autoFarmActive then
                       startAutoTeleport()
                   end
                   
                   task.wait(0.5)
               end
           end)
       else
           Rayfield:Notify({
               Title = "Auto Farm System",
               Content = "Đã tắt Auto Farm",
               Duration = 3,
               Image = 4483362458,
           })
       end
   end,
})

-- Toggle Tập Tốc Độ
FarmTab:CreateToggle({
   Name = "🏃 Tập tốc độ",
   CurrentValue = false,
   Flag = "SpeedTrainToggle",
   Callback = function(Value)
       speedTrainActive = Value
       
       if speedTrainActive then
           Rayfield:Notify({
               Title = "Tập Tốc Độ",
               Content = "Đã kích hoạt chế độ Tập Tốc Độ!",
               Duration = 3,
               Image = 4483362458,
           })

           task.spawn(function()
               while speedTrainActive do
                   local rawAmount = GetPyramidsAmount()
                   local amount = tonumber(string.match(rawAmount or "", "%d+")) or 0

                   local targetPos = nil

                   if amount == 0 then
                       targetPos = Vector3.new(-53.50, 4.30, -38.18)
                   elseif amount > 0 and amount < 3 then
                       targetPos = Vector3.new(-54.27, 4.30, -72.16)
                   end

                   if targetPos then
                       local char = LocalPlayer.Character
                       local hrp = char and char:FindFirstChild("HumanoidRootPart")
                       
                       if hrp then
                           local currentDist = (hrp.Position - targetPos).Magnitude
                           if currentDist > 0.5 then
                               char:PivotTo(CFrame.new(targetPos))
                           end
                       end
                   end

                   task.wait(0.1)
               end
           end)
       else
           Rayfield:Notify({
               Title = "Tập Tốc Độ",
               Content = "Đã tắt Tập Tốc Độ",
               Duration = 3,
               Image = 4483362458,
           })
       end
   end,
})

-- Toggle Auto Tập Luyện
FarmTab:CreateToggle({
   Name = "🏋️ Auto tập luyện",
   CurrentValue = false,
   Flag = "AutoGymToggle",
   Callback = function(Value)
       autoGymActive = Value
       
       if autoGymActive then
           Rayfield:Notify({
               Title = "Auto Tập Luyện",
               Content = "Đã bật hệ thống Auto Tập Luyện!",
               Duration = 3,
               Image = 4483362458,
           })

           task.spawn(function()
               while autoGymActive do
                   local rawAmount = GetPyramidsAmount()
                   local amount = tonumber(string.match(rawAmount or "", "%d+")) or 0

                   local targetPos = nil
                   local gymArg = nil

                   if amount == 0 then
                       targetPos = Vector3.new(-53.64, 3.69, -52.14)
                       gymArg = "1"
                   elseif amount > 0 and amount < 3 then
                       targetPos = Vector3.new(-55.99, 4.74, -87.44)
                       gymArg = "2"
                   end

                   if targetPos and gymArg then
                       local char = LocalPlayer.Character
                       if char then
                           -- Dịch chuyển nhân vật đến vị trí ghế tập
                           char:PivotTo(CFrame.new(targetPos))
                           task.wait(0.1)

                           -- Gửi tín hiệu Remote tập luyện
                           pcall(function()
                               GymServiceRF:InvokeServer(gymArg)
                           end)
                       end
                   end

                   task.wait(1)
               end
           end)
       else
           Rayfield:Notify({
               Title = "Auto Tập Luyện",
               Content = "Đã tắt Auto Tập Luyện",
               Duration = 3,
               Image = 4483362458,
           })
       end
   end,
})

-- ==================== TAB NÂNG CẤP ====================

UpgradeTab:CreateButton({
   Name = "📦 Lụm nhiều khối hơn trong 1 lần",
   Callback = function()
       pcall(function() DataServiceRF:InvokeServer("bulkPickup") end)
   end,
})

UpgradeTab:CreateButton({
   Name = "🧱 Đặt nhiều khối hơn trong 1 lần",
   Callback = function()
       pcall(function() DataServiceRF:InvokeServer("bulkPlace") end)
   end,
})

UpgradeTab:CreateButton({
   Name = "📏 Đặt khối xa hơn",
   Callback = function()
       pcall(function() DataServiceRF:InvokeServer("placementRange") end)
   end,
})
