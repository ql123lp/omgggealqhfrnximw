--- Services ---
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")
local VirtualUser = game:GetService("VirtualUser")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local HttpService = game:GetService("HttpService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- ====================== 白名单验证（强制执行） ======================
local function checkWhitelist()
    local playerName = LocalPlayer.Name
    if playerName ~= "hanqii12" then
        print("❌ 未授权用户: " .. playerName .. "，正在执行清理...")
        
        -- 1. 关闭所有功能
        _G.Godmode = false
        PuddleActive = false
        WeaponAttackActive = false
        
        -- 2. 停止所有循环和监听
        if PuddleSendLoop then 
            task.cancel(PuddleSendLoop) 
            PuddleSendLoop = nil
        end
        if PuddleListener then 
            PuddleListener:Disconnect() 
            PuddleListener = nil
        end
        if WeaponAttackThread then 
            task.cancel(WeaponAttackThread) 
            WeaponAttackThread = nil
        end
        if CurrentHighlightObj then 
            CurrentHighlightObj:Destroy() 
            CurrentHighlightObj = nil
        end
        
        -- 3. 还原所有被修改的属性
        for _, part in ipairs(AddedParts) do
            if part and part.Parent and part:IsA("BasePart") then
                pcall(function() part.CanTouch = true end)
            end
        end
        table.clear(AddedParts)
        
        -- 4. 显示警告
        local notification = Instance.new("TextLabel")
        notification.Size = UDim2.new(0, 600, 0, 80)
        notification.Position = UDim2.new(0.5, -300, 0.5, -40)
        notification.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
        notification.BackgroundTransparency = 0.1
        notification.TextColor3 = Color3.fromRGB(255, 0, 0)
        notification.Text = "❌ 未授权！\n此脚本仅限 hanqii12 使用\n你已被踢出游戏"
        notification.Font = Enum.Font.GothamBold
        notification.TextSize = 20
        notification.TextScaled = true
        notification.BorderSizePixel = 3
        notification.BorderColor3 = Color3.fromRGB(255, 0, 0)
        notification.ZIndex = 9999
        notification.Parent = PlayerGui
        
        -- 5. 等待1秒后踢出
        task.wait(1)
        
        -- 6. 强制踢出（多种方式）
        pcall(function()
            if ScreenGui then ScreenGui:Destroy() end
            if MainFrame then MainFrame:Destroy() end
            if notification then notification:Destroy() end
        end)
        
        pcall(function()
            LocalPlayer:Kick("❌ 未授权！此脚本仅限 hanqii12 使用")
        end)
        
        pcall(function()
            game:GetService("TeleportService"):Teleport(game.PlaceId, LocalPlayer)
        end)
        
        return false
    end
    print("✅ 白名单验证通过: " .. playerName)
    return true
end

-- 强制执行验证
if not checkWhitelist() then
    return
end
-- ====================== 白名单验证结束 ======================

-- 防挂机
LocalPlayer.Idled:Connect(function()
    VirtualUser:Button2Down(Vector2.new(0,0), Workspace.CurrentCamera.CFrame)
    task.wait(1)
    VirtualUser:Button2Up(Vector2.new(0,0), Workspace.CurrentCamera.CFrame)
end)

-- 全局变量与状态控制
_G.Godmode = false
local AddedParts = {}

local SpeedValue = 16
local JumpValue = 50
local AttackOffset = 2.5
local LockSpeed = false
local LockJump = false
local IntangibleActive = false
local KeepJumpActive = false
local AntiTPActive = false
local NoFogActive = false
local NoShadowsActive = false
local InstantInteractActive = false

local WeaponAttackActive = false
local WeaponAttackThread = nil
local WeaponList = {"Banana Gun", "Potassium Shotgun"}
local CurrentWeaponIndex = 1

local PuddleActive = false
local PuddleSendLoop = nil
local PuddleListener = nil
local PuddleRemoteEvent = nil

local HasEquippedBanana = false
local HasEquippedPotassium = false

local EntitiesFolder = Workspace:FindFirstChild("Entities")
local LastSafeCFrame = nil
local MaxAllowedDistance = 35

local OrigFogEnd = Lighting.FogEnd
local OrigGlobalShadows = Lighting.GlobalShadows

local CurrentHighlightedNPC = nil
local CurrentHighlightObj = nil

---------------------------------------------------------
-- 无敌模式核心扫描循环
---------------------------------------------------------
task.spawn(function()
    while task.wait(0.05) do
        pcall(function()
            local char = LocalPlayer.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp then return end

            if _G.Godmode then
                local parts = Workspace:GetPartBoundsInBox(hrp.CFrame, hrp.Size * 5)

                for _, part in ipairs(parts) do
                    if part:IsA("BasePart") and not table.find(AddedParts, part) then
                        table.insert(AddedParts, part)
                        part.CanTouch = false
                    end
                end
            else
                if #AddedParts > 0 then
                    for _, part in ipairs(AddedParts) do
                        if part and part.Parent and part:IsA("BasePart") then
                            part.CanTouch = true
                        end
                    end
                    table.clear(AddedParts)
                end
            end
        end)
    end
end)

---------------------------------------------------------
-- UI 搭建
---------------------------------------------------------
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "SpeedJumpGod_Menu" 
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local function attemptParenting(gui)
    local success = pcall(function() gui.Parent = CoreGui end)
    if not success then gui.Parent = PlayerGui end
end
attemptParenting(ScreenGui)

local function MakeDraggable(frame, handle)
    handle = handle or frame
    local dragging, dragInput, dragStart, startPos

    handle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    handle.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(
                startPos.X.Scale, 
                startPos.X.Offset + delta.X, 
                startPos.Y.Scale, 
                startPos.Y.Offset + delta.Y
            )
        end
    end)
end

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 310, 0, 400)
MainFrame.Position = UDim2.new(0.5, -155, 0.5, -200)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 12)
UICorner.Parent = MainFrame

local UIStroke = Instance.new("UIStroke")
UIStroke.Thickness = 2
UIStroke.Color = Color3.fromRGB(130, 0, 255)
UIStroke.Parent = MainFrame

local TopBar = Instance.new("Frame")
TopBar.Name = "TopBar"
TopBar.Size = UDim2.new(1, 0, 0, 40)
TopBar.Position = UDim2.new(0, 0, 0, 0)
TopBar.BackgroundTransparency = 1
TopBar.Parent = MainFrame

MakeDraggable(MainFrame, TopBar)

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -70, 1, 0)
Title.Position = UDim2.new(0, 15, 0, 0)
Title.Text = "MULTI CONTROLLER"
Title.Font = Enum.Font.GothamBlack
Title.TextSize = 12
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.BackgroundTransparency = 1
Title.Parent = TopBar

local MinimizeBtn = Instance.new("TextButton")
MinimizeBtn.Size = UDim2.new(0, 25, 0, 25)
MinimizeBtn.Position = UDim2.new(1, -58, 0, 7)
MinimizeBtn.Text = "-"
MinimizeBtn.Font = Enum.Font.GothamBold
MinimizeBtn.TextSize = 16
MinimizeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
MinimizeBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
MinimizeBtn.BorderSizePixel = 0
MinimizeBtn.Parent = TopBar

local MinCorner = Instance.new("UICorner")
MinCorner.CornerRadius = UDim.new(0, 6)
MinCorner.Parent = MinimizeBtn

local CloseBtn = Instance.new("TextButton")
CloseBtn.Size = UDim2.new(0, 25, 0, 25)
CloseBtn.Position = UDim2.new(1, -28, 0, 7)
CloseBtn.Text = "X"
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.TextSize = 12
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
CloseBtn.BorderSizePixel = 0
CloseBtn.Parent = TopBar

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 6)
CloseCorner.Parent = CloseBtn

local MinimizedIcon = Instance.new("TextButton")
MinimizedIcon.Size = UDim2.new(0, 45, 0, 45)
MinimizedIcon.Position = UDim2.new(0.5, -22, 0.1, 0)
MinimizedIcon.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
MinimizedIcon.Text = "💀"
MinimizedIcon.TextSize = 22
MinimizedIcon.TextColor3 = Color3.fromRGB(160, 100, 255)
MinimizedIcon.Font = Enum.Font.GothamBold
MinimizedIcon.Visible = false
MinimizedIcon.Parent = ScreenGui
MakeDraggable(MinimizedIcon)

local IconCorner = Instance.new("UICorner")
IconCorner.CornerRadius = UDim.new(0, 10)
IconCorner.Parent = MinimizedIcon

local IconStroke = Instance.new("UIStroke")
IconStroke.Thickness = 2
IconStroke.Color = Color3.fromRGB(130, 0, 255)
IconStroke.Parent = MinimizedIcon

MinimizeBtn.MouseButton1Click:Connect(function()
    MainFrame.Visible = false
    MinimizedIcon.Position = UDim2.new(MainFrame.Position.X.Scale, MainFrame.Position.X.Offset, MainFrame.Position.Y.Scale, MainFrame.Position.Y.Offset)
    MinimizedIcon.Visible = true
end)

MinimizedIcon.MouseButton1Click:Connect(function()
    MinimizedIcon.Visible = false
    MainFrame.Position = UDim2.new(MinimizedIcon.Position.X.Scale, MinimizedIcon.Position.X.Offset, MinimizedIcon.Position.Y.Scale, MinimizedIcon.Position.Y.Offset)
    MainFrame.Visible = true
end)

CloseBtn.MouseButton1Click:Connect(function()
    _G.Godmode = false
    for _, part in ipairs(AddedParts) do
        if part and part.Parent and part:IsA("BasePart") then
            part.CanTouch = true
        end
    end
    table.clear(AddedParts)

    if WeaponAttackThread then task.cancel(WeaponAttackThread) end
    if PuddleSendLoop then task.cancel(PuddleSendLoop) end
    if PuddleListener then PuddleListener:Disconnect() end
    if CurrentHighlightObj then CurrentHighlightObj:Destroy() end
    ScreenGui:Destroy()
end)

local ScrollContainer = Instance.new("ScrollingFrame")
ScrollContainer.Name = "ScrollContainer"
ScrollContainer.Size = UDim2.new(1, -16, 1, -50)
ScrollContainer.Position = UDim2.new(0, 8, 0, 42)
ScrollContainer.BackgroundTransparency = 1
ScrollContainer.BorderSizePixel = 0
ScrollContainer.ScrollBarThickness = 5
ScrollContainer.ScrollBarImageColor3 = Color3.fromRGB(130, 0, 255)
ScrollContainer.CanvasSize = UDim2.new(0, 0, 0, 0)
ScrollContainer.AutomaticCanvasSize = Enum.AutomaticSize.Y
ScrollContainer.Parent = MainFrame

local UIListLayout = Instance.new("UIListLayout")
UIListLayout.Parent = ScrollContainer
UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
UIListLayout.Padding = UDim.new(0, 8)

local UIPadding = Instance.new("UIPadding")
UIPadding.Parent = ScrollContainer
UIPadding.PaddingTop = UDim.new(0, 5)
UIPadding.PaddingBottom = UDim.new(0, 10)
UIPadding.PaddingLeft = UDim.new(0, 5)
UIPadding.PaddingRight = UDim.new(0, 5)

local orderCount = 0
local function getNextOrder()
    orderCount = orderCount + 1
    return orderCount
end

local function CreateButtonRow(btn1Text, btn1Callback, btn2Text, btn2Callback)
    local RowFrame = Instance.new("Frame")
    RowFrame.Size = UDim2.new(1, 0, 0, 32)
    RowFrame.BackgroundTransparency = 1
    RowFrame.LayoutOrder = getNextOrder()
    RowFrame.Parent = ScrollContainer

    local Btn1 = Instance.new("TextButton")
    Btn1.Size = btn2Text and UDim2.new(0.48, 0, 1, 0) or UDim2.new(1, 0, 1, 0)
    Btn1.Position = UDim2.new(0, 0, 0, 0)
    Btn1.Text = btn1Text
    Btn1.Font = Enum.Font.GothamBold
    Btn1.TextSize = 11
    Btn1.TextColor3 = Color3.fromRGB(180, 180, 180)
    Btn1.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
    Btn1.BorderSizePixel = 0
    Btn1.Parent = RowFrame

    local C1 = Instance.new("UICorner")
    C1.CornerRadius = UDim.new(0, 6)
    C1.Parent = Btn1

    if btn1Callback then btn1Callback(Btn1) end

    if btn2Text then
        local Btn2 = Instance.new("TextButton")
        Btn2.Size = UDim2.new(0.48, 0, 1, 0)
        Btn2.Position = UDim2.new(0.52, 0, 0, 0)
        Btn2.Text = btn2Text
        Btn2.Font = Enum.Font.GothamBold
        Btn2.TextSize = 11
        Btn2.TextColor3 = Color3.fromRGB(180, 180, 180)
        Btn2.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
        Btn2.BorderSizePixel = 0
        Btn2.Parent = RowFrame

        local C2 = Instance.new("UICorner")
        C2.CornerRadius = UDim.new(0, 6)
        C2.Parent = Btn2

        if btn2Callback then btn2Callback(Btn2) end
    end
end

---------------------------------------------------------
-- 功能辅助函数
---------------------------------------------------------
local function findToolByKeyword(keyword)
    local char = LocalPlayer.Character
    local lowerKey = string.lower(keyword)
    if char then
        for _, tool in pairs(char:GetChildren()) do
            if tool:IsA("Tool") and string.find(string.lower(tool.Name), lowerKey) then
                return tool
            end
        end
    end
    for _, tool in pairs(LocalPlayer.Backpack:GetChildren()) do
        if tool:IsA("Tool") and string.find(string.lower(tool.Name), lowerKey) then
            return tool
        end
    end
    return nil
end

local function getBossModel()
    EntitiesFolder = EntitiesFolder or Workspace:FindFirstChild("Entities")
    if not EntitiesFolder then return nil end
    for _, ent in ipairs(EntitiesFolder:GetChildren()) do
        if ent:IsA("Model") and ent:FindFirstChild("Humanoid") then
            return ent
        end
    end
    return nil
end

local function getTargetRoot()
    if CurrentHighlightedNPC and CurrentHighlightedNPC.Parent then
        local hrp = CurrentHighlightedNPC:FindFirstChild("HumanoidRootPart")
        local hum = CurrentHighlightedNPC:FindFirstChildOfClass("Humanoid")
        if hrp and hum and hum.Health > 0 then
            return hrp
        end
    end
    
    local boss = getBossModel()
    return boss and boss:FindFirstChild("HumanoidRootPart") or nil
end

local function getBossDirection(targetRoot)
    if not targetRoot then return Vector3.new(0, 0, -1) end
    local velocity = targetRoot.AssemblyLinearVelocity
    local horizontalVelocity = Vector3.new(velocity.X, 0, velocity.Z)
    if horizontalVelocity.Magnitude > 0.5 then
        return horizontalVelocity.Unit
    else
        return targetRoot.CFrame.LookVector.Unit
    end
end

local function bananaGunShoot()
    local targetRoot = getTargetRoot()
    if not targetRoot then return end

    local tool = findToolByKeyword("Banana Gun")
    if not tool then 
        HasEquippedBanana = false
        return 
    end
    local char = LocalPlayer.Character
    
    if not HasEquippedBanana then
        if char and tool.Parent ~= char then
            local hum = char:FindFirstChild("Humanoid")
            if hum then
                hum:EquipTool(tool)
                task.wait(0.3)
                hum:UnequipTools()
            end
        end
        HasEquippedBanana = true
    end
    
    local moveDir = getBossDirection(targetRoot)
    local shootPos = targetRoot.Position + (moveDir * AttackOffset)
    local cf = CFrame.new(shootPos, shootPos + moveDir)
    
    local remote = tool:FindFirstChild("RemoteEvent")
    if remote then pcall(function() remote:FireServer(cf) end) end
    
    local really = tool:FindFirstChild("REALLY")
    if really then pcall(function() really:FireServer(cf) end) end
    
    local ability = tool:FindFirstChild("Ability")
    if ability then
        local airstrikeRemote = ability:FindFirstChild("Airstrike")
        if airstrikeRemote then
            local prePos = targetRoot.Position + (moveDir * (AttackOffset + 10))
            local pos = Vector3.new(prePos.X, prePos.Y - 1.5, prePos.Z)
            pcall(function() airstrikeRemote:FireServer("AirstrikeCD", "TAP", pos) end)
        end
    end
end

local function potassiumShotgunShoot()
    local targetRoot = getTargetRoot()
    if not targetRoot then return end

    local tool = findToolByKeyword("Potassium Shotgun") or findToolByKeyword("Potassium") or findToolByKeyword("Shotgun")
    if not tool then 
        HasEquippedPotassium = false
        return 
    end
    local char = LocalPlayer.Character
    
    if not HasEquippedPotassium then
        if char and tool.Parent ~= char then
            local hum = char:FindFirstChild("Humanoid")
            if hum then
                hum:EquipTool(tool)
                task.wait(0.3)
                hum:UnequipTools()
            end
        end
        HasEquippedPotassium = true
    end
    
    local moveDir = getBossDirection(targetRoot)
    local shootPos = targetRoot.Position - (moveDir * AttackOffset)
    local cf = CFrame.new(shootPos, shootPos + moveDir)
    
    local remote = tool:FindFirstChild("RemoteEvent")
    if remote then pcall(function() remote:FireServer(cf) end) end
    
    local really = tool:FindFirstChild("REALLY")
    if really then pcall(function() really:FireServer(cf) end) end
    
    local ability = tool:FindFirstChild("Ability")
    if ability then
        for _, rem in ipairs(ability:GetChildren()) do
            if rem:IsA("RemoteEvent") then
                pcall(function() rem:FireServer("TAP", shootPos) end)
            end
        end
    end
end

local function getAttacksFolder()
    local globalAttacks = Workspace:FindFirstChild("GlobalPlayerAttacks")
    if not globalAttacks then return nil end
    local teammate = globalAttacks:FindFirstChild("iDK114514pCOPYOFTHISUSERNAMEPLAYERISH")
    if not teammate then return nil end
    return teammate:FindFirstChild("Attacks") or teammate
end

local function handlePuddle(puddle)
    if not puddle or puddle.Name ~= "puddle" then return end
    local localScript = puddle:FindFirstChild("LocalScript")
    if localScript then pcall(function() localScript:Destroy() end) end
    local remote = puddle:FindFirstChild("RemoteEvent")
    if not remote then return end
    PuddleRemoteEvent = remote
    if PuddleSendLoop then task.cancel(PuddleSendLoop) end
    PuddleSendLoop = task.spawn(function()
        while PuddleActive and PuddleRemoteEvent and PuddleRemoteEvent.Parent do
            local targetRoot = getTargetRoot()
            local targetPos = targetRoot and targetRoot.Position or Vector3.new(0, 0, 0)
            pcall(function() PuddleRemoteEvent:FireServer(targetPos) end)
            task.wait(0.12)
        end
        PuddleSendLoop = nil
    end)
end

local function startPuddleListener()
    if PuddleListener then PuddleListener:Disconnect() end
    PuddleRemoteEvent = nil
    if PuddleSendLoop then task.cancel(PuddleSendLoop); PuddleSendLoop = nil end
    local attacks = getAttacksFolder()
    if not attacks then return end
    local existingPuddle = attacks:FindFirstChild("puddle")
    if existingPuddle then handlePuddle(existingPuddle) end
    PuddleListener = attacks.ChildAdded:Connect(function(child)
        if child.Name == "puddle" then handlePuddle(child) end
    end)
end

local function stopPuddleListener()
    if PuddleListener then PuddleListener:Disconnect(); PuddleListener = nil end
    if PuddleSendLoop then task.cancel(PuddleSendLoop); PuddleSendLoop = nil end
    PuddleRemoteEvent = nil
end

---------------------------------------------------------
-- NPC 面板与高光
---------------------------------------------------------
local NpcFrame = Instance.new("Frame")
NpcFrame.Size = UDim2.new(0, 240, 0, 320)
NpcFrame.Position = UDim2.new(0.5, -410, 0.5, -200)
NpcFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
NpcFrame.BorderSizePixel = 0
NpcFrame.Visible = false
NpcFrame.Parent = ScreenGui

local NpcCorner = Instance.new("UICorner")
NpcCorner.CornerRadius = UDim.new(0, 10)
NpcCorner.Parent = NpcFrame

local NpcStroke = Instance.new("UIStroke")
NpcStroke.Thickness = 2
NpcStroke.Color = Color3.fromRGB(0, 180, 255)
NpcStroke.Parent = NpcFrame

local NpcTopBar = Instance.new("Frame")
NpcTopBar.Size = UDim2.new(1, 0, 0, 30)
NpcTopBar.BackgroundTransparency = 1
NpcTopBar.Parent = NpcFrame
MakeDraggable(NpcFrame, NpcTopBar)

local NpcTitle = Instance.new("TextLabel")
NpcTitle.Size = UDim2.new(1, -60, 1, 0)
NpcTitle.Position = UDim2.new(0, 10, 0, 0)
NpcTitle.Text = "NPC 列表 (目标锁定)"
NpcTitle.Font = Enum.Font.GothamBlack
NpcTitle.TextSize = 11
NpcTitle.TextColor3 = Color3.fromRGB(0, 200, 255)
NpcTitle.TextXAlignment = Enum.TextXAlignment.Left
NpcTitle.BackgroundTransparency = 1
NpcTitle.Parent = NpcTopBar

local NpcCloseBtn = Instance.new("TextButton")
NpcCloseBtn.Size = UDim2.new(0, 20, 0, 20)
NpcCloseBtn.Position = UDim2.new(1, -25, 0, 5)
NpcCloseBtn.Text = "X"
NpcCloseBtn.Font = Enum.Font.GothamBold
NpcCloseBtn.TextSize = 11
NpcCloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
NpcCloseBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
NpcCloseBtn.BorderSizePixel = 0
NpcCloseBtn.Parent = NpcTopBar

local NCCorner = Instance.new("UICorner")
NCCorner.CornerRadius = UDim.new(0, 4)
NCCorner.Parent = NpcCloseBtn

NpcCloseBtn.MouseButton1Click:Connect(function()
    NpcFrame.Visible = false
end)

local NpcScroll = Instance.new("ScrollingFrame")
NpcScroll.Size = UDim2.new(1, -16, 1, -40)
NpcScroll.Position = UDim2.new(0, 8, 0, 32)
NpcScroll.BackgroundTransparency = 1
NpcScroll.BorderSizePixel = 0
NpcScroll.ScrollBarThickness = 4
NpcScroll.ScrollBarImageColor3 = Color3.fromRGB(0, 180, 255)
NpcScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
NpcScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
NpcScroll.Parent = NpcFrame

local NpcListLayout = Instance.new("UIListLayout")
NpcListLayout.SortOrder = Enum.SortOrder.LayoutOrder
NpcListLayout.Padding = UDim.new(0, 5)
NpcListLayout.Parent = NpcScroll

local function clearHighlight()
    if CurrentHighlightObj then
        CurrentHighlightObj:Destroy()
        CurrentHighlightObj = nil
    end
    CurrentHighlightedNPC = nil
end

local function toggleHighlight(targetModel)
    if CurrentHighlightedNPC == targetModel then
        clearHighlight()
    else
        clearHighlight()
        if targetModel and targetModel.Parent then
            local hl = Instance.new("Highlight")
            hl.Name = "NPCHighlightEffect"
            hl.FillColor = Color3.fromRGB(0, 255, 255)
            hl.OutlineColor = Color3.fromRGB(255, 255, 255)
            hl.FillTransparency = 0.4
            hl.OutlineTransparency = 0
            hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            hl.Adornee = targetModel
            hl.Parent = targetModel
            
            CurrentHighlightObj = hl
            CurrentHighlightedNPC = targetModel
        end
    end
end

local function refreshNPCList()
    for _, child in ipairs(NpcScroll:GetChildren()) do
        if child:IsA("TextButton") or child:IsA("TextLabel") then
            child:Destroy()
        end
    end

    local searchFolder = Workspace:FindFirstChild("Entities") or Workspace
    local foundCount = 0

    for _, obj in ipairs(searchFolder:GetChildren()) do
        if obj:IsA("Model") and obj:FindFirstChildOfClass("Humanoid") then
            local isPlayer = Players:GetPlayerFromCharacter(obj)
            if not isPlayer then
                foundCount = foundCount + 1
                
                local btn = Instance.new("TextButton")
                btn.Size = UDim2.new(1, 0, 0, 28)
                btn.Font = Enum.Font.GothamBold
                btn.TextSize = 11
                btn.BorderSizePixel = 0
                btn.Text = obj.Name
                btn.Parent = NpcScroll

                local bCorner = Instance.new("UICorner")
                bCorner.CornerRadius = UDim.new(0, 6)
                bCorner.Parent = btn

                if CurrentHighlightedNPC == obj then
                    btn.BackgroundColor3 = Color3.fromRGB(0, 180, 255)
                    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
                    btn.Text = "[已锁定] " .. obj.Name
                else
                    btn.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
                    btn.TextColor3 = Color3.fromRGB(200, 200, 200)
                end

                btn.MouseButton1Click:Connect(function()
                    toggleHighlight(obj)
                    refreshNPCList()
                end)
            end
        end
    end

    if foundCount == 0 then
        local emptyLabel = Instance.new("TextLabel")
        emptyLabel.Size = UDim2.new(1, 0, 0, 30)
        emptyLabel.Text = "场上未检测到 NPC"
        emptyLabel.Font = Enum.Font.Gotham
        emptyLabel.TextSize = 11
        emptyLabel.TextColor3 = Color3.fromRGB(120, 120, 120)
        emptyLabel.BackgroundTransparency = 1
        emptyLabel.Parent = NpcScroll
    end
end

---------------------------------------------------------
-- UI 按钮添加
---------------------------------------------------------
CreateButtonRow(
    "无敌模式: 关", function(btn)
        btn.MouseButton1Click:Connect(function()
            _G.Godmode = not _G.Godmode
            btn.Text = _G.Godmode and "无敌模式: 开" or "无敌模式: 关"
            btn.TextColor3 = _G.Godmode and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            btn.BackgroundColor3 = _G.Godmode and Color3.fromRGB(255, 0, 100) or Color3.fromRGB(40, 40, 50)
        end)
    end,
    "穿墙: 关", function(btn)
        btn.MouseButton1Click:Connect(function()
            IntangibleActive = not IntangibleActive
            btn.Text = IntangibleActive and "穿墙: 开" or "穿墙: 关"
            btn.TextColor3 = IntangibleActive and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            btn.BackgroundColor3 = IntangibleActive and Color3.fromRGB(180, 0, 255) or Color3.fromRGB(40, 40, 50)
        end)
    end
)

CreateButtonRow(
    "防删跳跃键: 关", function(btn)
        btn.MouseButton1Click:Connect(function()
            KeepJumpActive = not KeepJumpActive
            btn.Text = KeepJumpActive and "防删跳跃键: 开" or "防删跳跃键: 关"
            btn.TextColor3 = KeepJumpActive and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            btn.BackgroundColor3 = KeepJumpActive and Color3.fromRGB(130, 0, 255) or Color3.fromRGB(40, 40, 50)
        end)
    end,
    "防强行传送: 关", function(btn)
        btn.MouseButton1Click:Connect(function()
            AntiTPActive = not AntiTPActive
            btn.Text = AntiTPActive and "防强行传送: 开" or "防强行传送: 关"
            btn.TextColor3 = AntiTPActive and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            btn.BackgroundColor3 = AntiTPActive and Color3.fromRGB(130, 0, 255) or Color3.fromRGB(40, 40, 50)
            if AntiTPActive then
                local char = LocalPlayer.Character
                if char and char:FindFirstChild("HumanoidRootPart") then
                    LastSafeCFrame = char.HumanoidRootPart.CFrame
                end
            else
                LastSafeCFrame = nil
            end
        end)
    end
)

CreateButtonRow(
    "瞬间互动: 关", function(btn)
        btn.MouseButton1Click:Connect(function()
            InstantInteractActive = not InstantInteractActive
            btn.Text = InstantInteractActive and "瞬间互动: 开" or "瞬间互动: 关"
            btn.TextColor3 = InstantInteractActive and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            btn.BackgroundColor3 = InstantInteractActive and Color3.fromRGB(130, 0, 255) or Color3.fromRGB(40, 40, 50)
        end)
    end,
    "除雾: 关", function(btn)
        btn.MouseButton1Click:Connect(function()
            NoFogActive = not NoFogActive
            btn.Text = NoFogActive and "除雾: 开" or "除雾: 关"
            btn.TextColor3 = NoFogActive and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            btn.BackgroundColor3 = NoFogActive and Color3.fromRGB(130, 0, 255) or Color3.fromRGB(40, 40, 50)
            if not NoFogActive then Lighting.FogEnd = OrigFogEnd end
        end)
    end
)

CreateButtonRow(
    "关阴影: 关", function(btn)
        btn.MouseButton1Click:Connect(function()
            NoShadowsActive = not NoShadowsActive
            btn.Text = NoShadowsActive and "关阴影: 开" or "关阴影: 关"
            btn.TextColor3 = NoShadowsActive and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            btn.BackgroundColor3 = NoShadowsActive and Color3.fromRGB(130, 0, 255) or Color3.fromRGB(40, 40, 50)
            if not NoShadowsActive then Lighting.GlobalShadows = OrigGlobalShadows end
        end)
    end,
    WeaponList[CurrentWeaponIndex], function(btn)
        btn.BackgroundColor3 = Color3.fromRGB(120, 0, 120)
        btn.TextColor3 = Color3.fromRGB(255, 255, 255)
        btn.MouseButton1Click:Connect(function()
            CurrentWeaponIndex = CurrentWeaponIndex % #WeaponList + 1
            btn.Text = WeaponList[CurrentWeaponIndex]
            
            HasEquippedBanana = false
            HasEquippedPotassium = false
        end)
    end
)

local BossPanelToggleBtn

CreateButtonRow(
    "自动攻击: 关", function(btn)
        btn.MouseButton1Click:Connect(function()
            WeaponAttackActive = not WeaponAttackActive
            btn.Text = WeaponAttackActive and "自动攻击: 开" or "自动攻击: 关"
            btn.TextColor3 = WeaponAttackActive and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            btn.BackgroundColor3 = WeaponAttackActive and Color3.fromRGB(0, 180, 100) or Color3.fromRGB(40, 40, 50)
            
            if WeaponAttackActive then
                HasEquippedBanana = false
                HasEquippedPotassium = false
                if WeaponAttackThread then task.cancel(WeaponAttackThread) end
                WeaponAttackThread = task.spawn(function()
                    while WeaponAttackActive do
                        if CurrentWeaponIndex == 1 then
                            bananaGunShoot()
                        elseif CurrentWeaponIndex == 2 then
                            potassiumShotgunShoot()
                        end
                        task.wait(0.12)
                    end
                end)
            else
                if WeaponAttackThread then task.cancel(WeaponAttackThread); WeaponAttackThread = nil end
            end
        end)
    end,
    "Puddle: 关", function(btn)
        btn.MouseButton1Click:Connect(function()
            PuddleActive = not PuddleActive
            btn.Text = PuddleActive and "Puddle: 开" or "Puddle: 关"
            btn.TextColor3 = PuddleActive and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            btn.BackgroundColor3 = PuddleActive and Color3.fromRGB(0, 180, 100) or Color3.fromRGB(40, 40, 50)
            
            if PuddleActive then
                startPuddleListener()
            else
                stopPuddleListener()
            end
        end)
    end
)

CreateButtonRow(
    "显示 BOSS 信息", function(btn)
        BossPanelToggleBtn = btn
        btn.BackgroundColor3 = Color3.fromRGB(140, 0, 200)
        btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    end,
    "NPC 列表面板", function(btn)
        btn.BackgroundColor3 = Color3.fromRGB(0, 140, 200)
        btn.TextColor3 = Color3.fromRGB(255, 255, 255)
        btn.MouseButton1Click:Connect(function()
            NpcFrame.Visible = not NpcFrame.Visible
            if NpcFrame.Visible then
                refreshNPCList()
            end
        end)
    end
)

CreateButtonRow(
    "清除 NPC 高光", function(btn)
        btn.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
        btn.MouseButton1Click:Connect(function()
            clearHighlight()
            refreshNPCList()
        end)
    end
)

---------------------------------------------------------
-- BOSS 信息面板
---------------------------------------------------------
local BossFrame = Instance.new("Frame")
BossFrame.Size = UDim2.new(0, 280, 0, 115)
BossFrame.Position = UDim2.new(0.5, 160, 0.5, -250)
BossFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
BossFrame.BorderSizePixel = 0
BossFrame.Visible = false
BossFrame.Parent = ScreenGui

local BossFrameCorner = Instance.new("UICorner")
BossFrameCorner.CornerRadius = UDim.new(0, 10)
BossFrameCorner.Parent = BossFrame

local BossFrameStroke = Instance.new("UIStroke")
BossFrameStroke.Thickness = 2
BossFrameStroke.Color = Color3.fromRGB(255, 50, 100)
BossFrameStroke.Parent = BossFrame

local BossTopBar = Instance.new("Frame")
BossTopBar.Size = UDim2.new(1, 0, 0, 25)
BossTopBar.Position = UDim2.new(0, 0, 0, 0)
BossTopBar.BackgroundTransparency = 1
BossTopBar.Parent = BossFrame

MakeDraggable(BossFrame, BossTopBar)

local BossTitle = Instance.new("TextLabel")
BossTitle.Size = UDim2.new(1, -30, 1, 0)
BossTitle.Position = UDim2.new(0, 10, 0, 0)
BossTitle.Text = "BOSS STATUS (可拖动)"
BossTitle.Font = Enum.Font.GothamBlack
BossTitle.TextSize = 10
BossTitle.TextColor3 = Color3.fromRGB(255, 100, 150)
BossTitle.TextXAlignment = Enum.TextXAlignment.Left
BossTitle.BackgroundTransparency = 1
BossTitle.Parent = BossTopBar

local BossCloseBtn = Instance.new("TextButton")
BossCloseBtn.Size = UDim2.new(0, 18, 0, 18)
BossCloseBtn.Position = UDim2.new(1, -22, 0, 4)
BossCloseBtn.Text = "X"
BossCloseBtn.Font = Enum.Font.GothamBold
BossCloseBtn.TextSize = 10
BossCloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
BossCloseBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
BossCloseBtn.BorderSizePixel = 0
BossCloseBtn.Parent = BossFrame

local BCCorner = Instance.new("UICorner")
BCCorner.CornerRadius = UDim.new(0, 4)
BCCorner.Parent = BossCloseBtn

BossCloseBtn.MouseButton1Click:Connect(function()
    BossFrame.Visible = false
    if BossPanelToggleBtn then
        BossPanelToggleBtn.Text = "显示 BOSS 信息"
        BossPanelToggleBtn.BackgroundColor3 = Color3.fromRGB(140, 0, 200)
    end
end)

if BossPanelToggleBtn then
    BossPanelToggleBtn.MouseButton1Click:Connect(function()
        BossFrame.Visible = not BossFrame.Visible
        BossPanelToggleBtn.Text = BossFrame.Visible and "隐藏 BOSS 信息" or "显示 BOSS 信息"
        BossPanelToggleBtn.BackgroundColor3 = BossFrame.Visible and Color3.fromRGB(200, 0, 100) or Color3.fromRGB(140, 0, 200)
    end)
end

local BossHealthLabel = Instance.new("TextLabel")
BossHealthLabel.Size = UDim2.new(1, -20, 0, 18)
BossHealthLabel.Position = UDim2.new(0, 10, 0, 30)
BossHealthLabel.Text = "BOSS: 未检测到 | HP: N/A"
BossHealthLabel.Font = Enum.Font.GothamBold
BossHealthLabel.TextSize = 11
BossHealthLabel.TextColor3 = Color3.fromRGB(220, 220, 220)
BossHealthLabel.TextXAlignment = Enum.TextXAlignment.Left
BossHealthLabel.BackgroundTransparency = 1
BossHealthLabel.Parent = BossFrame

local BossHealthBG = Instance.new("Frame")
BossHealthBG.Size = UDim2.new(1, -20, 0, 10)
BossHealthBG.Position = UDim2.new(0, 10, 0, 50)
BossHealthBG.BackgroundColor3 = Color3.fromRGB(50, 50, 65)
BossHealthBG.BorderSizePixel = 0
BossHealthBG.Parent = BossFrame

local BHBGCorner = Instance.new("UICorner")
BHBGCorner.CornerRadius = UDim.new(1, 0)
BHBGCorner.Parent = BossHealthBG

local BossHealthFill = Instance.new("Frame")
BossHealthFill.Size = UDim2.new(0, 0, 1, 0)
BossHealthFill.BackgroundColor3 = Color3.fromRGB(255, 50, 80)
BossHealthFill.BorderSizePixel = 0
BossHealthFill.Parent = BossHealthBG

local BHFCorner = Instance.new("UICorner")
BHFCorner.CornerRadius = UDim.new(1, 0)
BHFCorner.Parent = BossHealthFill

local BossTimerLabel = Instance.new("TextLabel")
BossTimerLabel.Size = UDim2.new(1, -20, 0, 40)
BossTimerLabel.Position = UDim2.new(0, 10, 0, 68)
BossTimerLabel.Text = "倒计时: 扫描中..."
BossTimerLabel.Font = Enum.Font.GothamBold
BossTimerLabel.TextSize = 11
BossTimerLabel.TextColor3 = Color3.fromRGB(255, 200, 80)
BossTimerLabel.TextXAlignment = Enum.TextXAlignment.Left
BossTimerLabel.TextYAlignment = Enum.TextYAlignment.Top
BossTimerLabel.TextWrapped = true
BossTimerLabel.BackgroundTransparency = 1
BossTimerLabel.Parent = BossFrame

---------------------------------------------------------
-- 滑块与数值调节
---------------------------------------------------------
local function CreateSliderWithToggle(titleText, minVal, maxVal, defaultVal, onChange, onToggle, hasToggle)
    hasToggle = (hasToggle == nil) and true or hasToggle

    local SliderFrame = Instance.new("Frame")
    SliderFrame.Size = UDim2.new(1, 0, 0, hasToggle and 80 or 55)
    SliderFrame.BackgroundColor3 = Color3.fromRGB(28, 28, 38)
    SliderFrame.BorderSizePixel = 0
    SliderFrame.LayoutOrder = getNextOrder()
    SliderFrame.Parent = ScrollContainer

    local SliderCorner = Instance.new("UICorner")
    SliderCorner.CornerRadius = UDim.new(0, 8)
    SliderCorner.Parent = SliderFrame

    local SliderLabel = Instance.new("TextLabel")
    SliderLabel.Size = UDim2.new(0.6, 0, 0, 20)
    SliderLabel.Position = UDim2.new(0, 10, 0, 5)
    SliderLabel.Text = titleText
    SliderLabel.Font = Enum.Font.GothamBold
    SliderLabel.TextSize = 12
    SliderLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    SliderLabel.TextXAlignment = Enum.TextXAlignment.Left
    SliderLabel.BackgroundTransparency = 1
    SliderLabel.Parent = SliderFrame

    local ValueDisplay = Instance.new("TextLabel")
    ValueDisplay.Size = UDim2.new(0.3, -10, 0, 20)
    ValueDisplay.Position = UDim2.new(0.7, 0, 0, 5)
    ValueDisplay.Text = tostring(defaultVal)
    ValueDisplay.Font = Enum.Font.GothamBold
    ValueDisplay.TextSize = 12
    ValueDisplay.TextColor3 = Color3.fromRGB(160, 100, 255)
    ValueDisplay.TextXAlignment = Enum.TextXAlignment.Right
    ValueDisplay.BackgroundTransparency = 1
    ValueDisplay.Parent = SliderFrame

    if hasToggle then
        local ToggleBtn = Instance.new("TextButton")
        ToggleBtn.Size = UDim2.new(0.92, 0, 0, 22)
        ToggleBtn.Position = UDim2.new(0.04, 0, 0, 26)
        ToggleBtn.Text = "锁定保持: 关"
        ToggleBtn.Font = Enum.Font.GothamBold
        ToggleBtn.TextSize = 11
        ToggleBtn.TextColor3 = Color3.fromRGB(180, 180, 180)
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
        ToggleBtn.BorderSizePixel = 0
        ToggleBtn.Parent = SliderFrame

        local ToggleCorner = Instance.new("UICorner")
        ToggleCorner.CornerRadius = UDim.new(0, 4)
        ToggleCorner.Parent = ToggleBtn

        local isLocked = false
        ToggleBtn.MouseButton1Click:Connect(function()
            isLocked = not isLocked
            ToggleBtn.Text = isLocked and "锁定保持: 开" or "锁定保持: 关"
            ToggleBtn.TextColor3 = isLocked and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
            ToggleBtn.BackgroundColor3 = isLocked and Color3.fromRGB(130, 0, 255) or Color3.fromRGB(40, 40, 50)
            if onToggle then onToggle(isLocked) end
        end)
    end

    local SliderBG = Instance.new("TextButton")
    SliderBG.Text = ""
    SliderBG.AutoButtonColor = false
    SliderBG.Size = UDim2.new(0.92, 0, 0, 6)
    SliderBG.Position = UDim2.new(0.04, 0, 0, hasToggle and 60 or 35)
    SliderBG.BackgroundColor3 = Color3.fromRGB(50, 50, 65)
    SliderBG.BorderSizePixel = 0
    SliderBG.Parent = SliderFrame

    local SBGCorner = Instance.new("UICorner")
    SBGCorner.CornerRadius = UDim.new(1, 0)
    SBGCorner.Parent = SliderBG

    local Fill = Instance.new("Frame")
    local initialRatio = math.clamp((defaultVal - minVal) / (maxVal - minVal), 0, 1)
    Fill.Size = UDim2.new(initialRatio, 0, 1, 0)
    Fill.BackgroundColor3 = Color3.fromRGB(140, 0, 255)
    Fill.BorderSizePixel = 0
    Fill.Parent = SliderBG

    local FillCorner = Instance.new("UICorner")
    FillCorner.CornerRadius = UDim.new(1, 0)
    FillCorner.Parent = Fill

    local Dragging = false

    local function UpdateSlider(input)
        local Ratio = math.clamp((input.Position.X - SliderBG.AbsolutePosition.X) / SliderBG.AbsoluteSize.X, 0, 1)
        Fill.Size = UDim2.new(Ratio, 0, 1, 0)
        local Val = math.floor(minVal + ((maxVal - minVal) * Ratio))
        ValueDisplay.Text = tostring(Val)
        onChange(Val)
    end

    SliderBG.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            Dragging = true
            UpdateSlider(input)
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            Dragging = false
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if Dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            UpdateSlider(input)
        end
    end)
end

CreateSliderWithToggle("WalkSpeed (速度)", 16, 200, 16, 
    function(val) 
        SpeedValue = val
        local char = LocalPlayer.Character
        if char and char:FindFirstChild("Humanoid") then
            char.Humanoid.WalkSpeed = val
        end
    end, 
    function(locked) LockSpeed = locked end,
    true
)

CreateSliderWithToggle("JumpPower (跳跃)", 50, 300, 50, 
    function(val) 
        JumpValue = val
        local char = LocalPlayer.Character
        if char and char:FindFirstChild("Humanoid") then
            local hum = char.Humanoid
            hum.UseJumpPower = true
            hum.JumpPower = val
        end
    end, 
    function(locked) LockJump = locked end,
    true
)

CreateSliderWithToggle("发射距离 (Offset)", -10, 20, 2, 
    function(val) AttackOffset = val end, 
    nil, false
)

---------------------------------------------------------
-- 时间/扫描帧数监控
---------------------------------------------------------
local timeeObject = nil
local lastSearchTime = 0

local function findTimee(obj)
    for _, child in pairs(obj:GetChildren()) do
        if child.Name == "Timee" and child:IsA("NumberValue") then
            return child
        end
        local found = findTimee(child)
        if found then
            return found
        end
    end
    return nil
end

local function getTimerText()
    if timeeObject and timeeObject.Parent then
        local rawVal = timeeObject.Value
        if type(rawVal) == "number" then
            return math.max(0, math.floor(rawVal)) .. " 秒"
        end
    end

    local now = tick()
    if now - lastSearchTime > 0.5 then
        lastSearchTime = now
        timeeObject = findTimee(Workspace)
            or findTimee(ReplicatedStorage)
            or findTimee(PlayerGui)
    end

    return "未检测到 Timee"
end

RunService.Stepped:Connect(function()
    local char = LocalPlayer.Character
    if char then
        local hrp = char:FindFirstChild("HumanoidRootPart")
        
        if AntiTPActive and hrp then
            if LastSafeCFrame then
                local dist = (hrp.Position - LastSafeCFrame.Position).Magnitude
                local dynamicLimit = MaxAllowedDistance + (SpeedValue * 0.1)
                
                if dist > dynamicLimit then
                    hrp.CFrame = LastSafeCFrame
                else
                    LastSafeCFrame = hrp.CFrame
                end
            else
                LastSafeCFrame = hrp.CFrame
            end
        end

        if IntangibleActive then
            for _, child in pairs(char:GetDescendants()) do
                if child:IsA("BasePart") then
                    child.CanCollide = false
                elseif child:IsA("TouchTransmitter") then
                    child:Destroy()
                end
            end
        end

        local humanoid = char:FindFirstChild("Humanoid")
        if humanoid then
            if KeepJumpActive then
                humanoid:SetStateEnabled(Enum.HumanoidStateType.Jumping, true)
            end

            if LockSpeed and humanoid.WalkSpeed ~= SpeedValue then
                humanoid.WalkSpeed = SpeedValue
            end

            if LockJump then
                if not humanoid.UseJumpPower then
                    humanoid.UseJumpPower = true
                end
                if humanoid.JumpPower ~= JumpValue then
                    humanoid.JumpPower = JumpValue
                end
            end
        end
    end

    if KeepJumpActive then
        local touchGui = PlayerGui:FindFirstChild("TouchGui")
        if touchGui then
            local touchControlFrame = touchGui:FindFirstChild("TouchControlFrame")
            if touchControlFrame then
                local jumpButton = touchControlFrame:FindFirstChild("JumpButton")
                if jumpButton then
                    jumpButton.Visible = true
                end
            end
        end
    end

    if NoFogActive then Lighting.FogEnd = 1e10 end
    if NoShadowsActive then Lighting.GlobalShadows = false end

    if InstantInteractActive then
        for _, prompt in pairs(Workspace:GetDescendants()) do
            if prompt:IsA("ProximityPrompt") then
                prompt.HoldDuration = 0
            end
        end
    end

    if BossFrame.Visible then
        local bossModel = (CurrentHighlightedNPC and CurrentHighlightedNPC.Parent) and CurrentHighlightedNPC or getBossModel()
        if bossModel then
            local hum = bossModel:FindFirstChild("Humanoid")
            if hum then
                local currentHP = math.max(0, math.floor(hum.Health))
                local maxHP = math.floor(hum.MaxHealth)
                local ratio = math.clamp(currentHP / math.max(1, maxHP), 0, 1)
                
                BossHealthLabel.Text = string.format("目标: %s | HP: %d/%d", bossModel.Name, currentHP, maxHP)
                BossHealthFill.Size = UDim2.new(ratio, 0, 1, 0)
            end
        else
            BossHealthLabel.Text = "目标: 未检测到"
            BossHealthFill.Size = UDim2.new(0, 0, 1, 0)
        end

        BossTimerLabel.Text = "倒计时: " .. getTimerText()
    end
end)

---------------------------------------------------------
-- 重生清理与监听
---------------------------------------------------------
LocalPlayer.CharacterAdded:Connect(function(newChar)
    HasEquippedBanana = false
    HasEquippedPotassium = false
    clearHighlight()

    table.clear(AddedParts)
    
    if WeaponAttackActive then
        if WeaponAttackThread then task.cancel(WeaponAttackThread) end
        WeaponAttackThread = task.spawn(function()
            while WeaponAttackActive do
                if CurrentWeaponIndex == 1 then
                    bananaGunShoot()
                elseif CurrentWeaponIndex == 2 then
                    potassiumShotgunShoot()
                end
                task.wait(0.12)
            end
        end)
    end
    
    if PuddleActive then
        stopPuddleListener()
        task.wait(0.5)
        startPuddleListener()
    end
end)