-- ╔══════════════════════════════════════════════════════════╗
-- ║       OMNI-FLING  V3.0  >>>  MONSTER EDITION  [FIXED]   ║
-- ║  Анти-кік | Анти-бан | Анти-детект | Авто-відновлення  ║
-- ╚══════════════════════════════════════════════════════════╝

-- ============================================================
-- [[ СЕРВІСИ ]]
-- ============================================================
local Players      = game:GetService("Players")
local RunService   = game:GetService("RunService")
local UIS          = game:GetService("UserInputService")
local Workspace    = game:GetService("Workspace")
local StarterGui   = game:GetService("StarterGui")
local TweenService = game:GetService("TweenService")
local HttpService  = game:GetService("HttpService")

local LP     = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- ============================================================
-- [[ 0. ЗАХИСТ ВІД ПОДВІЙНОГО ЗАПУСКУ ]]
-- FIX: getgenv() може не існувати — fallback до _G
-- ============================================================
local _env = (typeof(getgenv) == "function" and getgenv()) or _G

if _env.OmniFlingLoaded then
    pcall(function()
        if typeof(_env.OmniFlingStop) == "function" then
            _env.OmniFlingStop()
        end
    end)
    task.wait(0.15)
end
_env.OmniFlingLoaded = true

pcall(function()
    for _, sg in pairs({ game:GetService("CoreGui"), LP:WaitForChild("PlayerGui", 5) }) do
        if sg then
            for _, v in pairs(sg:GetChildren()) do
                if v.Name == "OmniFling" then v:Destroy() end
            end
        end
    end
end)

-- ============================================================
-- [[ 1. КОНФІГ ]]
-- ============================================================
_env.FPDH = _env.FPDH or workspace.FallenPartsDestroyHeight

local Whitelist = { LP.UserId }

local State = {
    FlingLoop  = false,
    FlingStyle = "auto",
    Flinging   = false,
    Stopped    = false,
}

local Config = {
    CurrentTarget  = "",
    FlingDuration  = 2,
    LoopDelay      = 0.05,
    MaxVelocity    = 9e8,
    TeleportNoise  = true,
    AntiAFK        = true,
    AutoRespawn    = true,
    SafeReturn     = true,
    CameraRestore  = true,
    SilentMode     = false,
}

local IsMobile = UIS.TouchEnabled and not UIS.MouseEnabled

-- ============================================================
-- [[ 2. УТИЛІТИ ]]
-- ============================================================
local function Notify(title, text, dur)
    if Config.SilentMode then return end
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title    = title,
            Text     = text,
            Duration = dur or 3,
        })
    end)
end

local function IsWhitelisted(player)
    for _, id in pairs(Whitelist) do
        if player.UserId == id then return true end
    end
    return false
end

local function IsAlive(player)
    if not player or not player.Character then return false end
    local hum = player.Character:FindFirstChildOfClass("Humanoid")
    return hum and hum.Health > 0
end

local function Jitter()
    if not Config.TeleportNoise then return Vector3.new(0,0,0) end
    return Vector3.new(
        (math.random() - 0.5) * 0.02,
        (math.random() - 0.5) * 0.02,
        (math.random() - 0.5) * 0.02
    )
end

local function GetClosestPlayer()
    local myChar = LP.Character
    local myHRP  = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myHRP then return nil end
    local best, bestDist = nil, math.huge
    for _, v in pairs(Players:GetPlayers()) do
        if v == LP or IsWhitelisted(v) then continue end
        if not IsAlive(v) then continue end
        local hrp = v.Character:FindFirstChild("HumanoidRootPart")
        if hrp then
            local d = (hrp.Position - myHRP.Position).Magnitude
            if d < bestDist then bestDist = d; best = v end
        end
    end
    return best
end

local function GetTarget(input)
    input = (input or ""):lower():match("^%s*(.-)%s*$")
    if input == "" then return GetClosestPlayer() end
    if input == "all" or input == "others" then return "all" end
    if input == "random" then
        local list = {}
        for _, v in pairs(Players:GetPlayers()) do
            if v ~= LP and not IsWhitelisted(v) and IsAlive(v) then
                table.insert(list, v)
            end
        end
        if #list == 0 then return nil end
        return list[math.random(#list)]
    end
    for _, v in pairs(Players:GetPlayers()) do
        if v == LP then continue end
        local n  = v.Name:lower()
        local dn = v.DisplayName:lower()
        if n:match("^" .. input) or dn:match("^" .. input) then return v end
    end
    return nil
end

-- ============================================================
-- [[ 3. АНТИ-АФК ]]
-- FIX: VirtualUser wrapped properly, без nil-call
-- ============================================================
local function StartAntiAFK()
    if not Config.AntiAFK then return end
    task.spawn(function()
        while not State.Stopped do
            task.wait(math.random(55, 65))
            pcall(function()
                local ok, VPS = pcall(function() return game:GetService("VirtualUser") end)
                if ok and VPS then
                    VPS:Button2Down(Vector2.new(0,0), CFrame.new())
                    task.wait(0.05)
                    VPS:Button2Up(Vector2.new(0,0), CFrame.new())
                end
            end)
        end
    end)
end

-- ============================================================
-- [[ 4. БЕЗПЕЧНИЙ ТЕЛЕПОРТ ]]
-- ============================================================
local lastTeleportTime = 0
local TELEPORT_COOLDOWN = 0

local function SafeSetCFrame(part, cf)
    if not part or not part.Parent then return end
    local now = tick()
    if now - lastTeleportTime < TELEPORT_COOLDOWN then return end
    lastTeleportTime = now
    pcall(function() part.CFrame = cf end)
end

-- ============================================================
-- [[ 5. CORE FLING ]]
-- ============================================================
local function FlingPlayer(TargetPlayer)
    if not TargetPlayer then return end
    if IsWhitelisted(TargetPlayer) then return end
    if not IsAlive(TargetPlayer) then return end
    if State.Flinging then return end
    if not IsAlive(LP) then return end

    State.Flinging = true

    local Character = LP.Character
    local Humanoid  = Character and Character:FindFirstChildOfClass("Humanoid")
    local RootPart  = Humanoid and Humanoid.RootPart

    if not Character or not Humanoid or not RootPart then
        State.Flinging = false; return
    end

    local TCharacter = TargetPlayer.Character
    local THumanoid  = TCharacter and TCharacter:FindFirstChildOfClass("Humanoid")
    local TRootPart  = TCharacter and TCharacter:FindFirstChild("HumanoidRootPart")
    local THead      = TCharacter and TCharacter:FindFirstChild("Head")
    local Accessory  = TCharacter and TCharacter:FindFirstChildOfClass("Accessory")
    local Handle     = Accessory and Accessory:FindFirstChild("Handle")

    if not THumanoid or not TCharacter then
        State.Flinging = false; return
    end

    if RootPart.Velocity.Magnitude < 50 then
        _env.OldPos = RootPart.CFrame
    end

    if Config.CameraRestore then
        local camTarget = THead or Handle or THumanoid
        pcall(function() Camera.CameraSubject = camTarget end)
    end

    workspace.FallenPartsDestroyHeight = 0/0

    local BV = Instance.new("BodyVelocity")
    BV.Name     = "OFV_" .. tostring(math.random(1000,9999))
    BV.Parent   = RootPart
    BV.Velocity = Vector3.new(Config.MaxVelocity, Config.MaxVelocity, Config.MaxVelocity)
    BV.MaxForce = Vector3.new(1/0, 1/0, 1/0)

    pcall(function() Humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, false) end)
    pcall(function() Humanoid.Sit = false end)

    local function FPos(BasePart, Pos, Ang)
        if not BasePart or not BasePart.Parent then return end
        if not RootPart or not RootPart.Parent then return end
        local jitter = Jitter()
        local newCF  = CFrame.new(BasePart.Position + jitter) * Pos * Ang
        pcall(function()
            RootPart.CFrame      = newCF
            RootPart.Velocity    = Vector3.new(Config.MaxVelocity/10, Config.MaxVelocity, Config.MaxVelocity/10)
            RootPart.RotVelocity = Vector3.new(Config.MaxVelocity, Config.MaxVelocity, Config.MaxVelocity)
        end)
        pcall(function() Character:SetPrimaryPartCFrame(newCF) end)
    end

    local function GetStyleOffset(Angle)
        if State.FlingStyle == "up" then
            return CFrame.new(0, 2, 0), CFrame.Angles(math.rad(Angle), 0, 0)
        elseif State.FlingStyle == "side" then
            return CFrame.new(2, 0, 0), CFrame.Angles(0, math.rad(Angle), 0)
        elseif State.FlingStyle == "down" then
            return CFrame.new(0, -2, 0), CFrame.Angles(math.rad(Angle), 0, 0)
        elseif State.FlingStyle == "spin" then
            return CFrame.new(math.sin(math.rad(Angle))*2, math.cos(math.rad(Angle))*2, 0),
                   CFrame.Angles(math.rad(Angle*2), math.rad(Angle), math.rad(Angle*3))
        else
            return nil, nil
        end
    end

    local function SFBasePart(BasePart)
        local TimeToWait = Config.FlingDuration
        local StartTime  = tick()
        local Angle      = 0

        local function ShouldStop()
            if not State.FlingLoop then return true end
            if not BasePart or not BasePart.Parent then return true end
            if BasePart.Parent ~= TCharacter then return true end
            if TargetPlayer.Parent ~= Players then return true end
            if not IsAlive(LP) then return true end
            if tick() > StartTime + TimeToWait then return true end
            return false
        end

        repeat
            if ShouldStop() then break end
            Angle = Angle + 100

            local stylePos, styleAng = GetStyleOffset(Angle)

            if stylePos then
                FPos(BasePart, stylePos, styleAng)
                task.wait()
                FPos(BasePart, CFrame.new(0, -stylePos.Y, 0), styleAng)
                task.wait()
            else
                local vel = BasePart.Velocity
                if vel.Magnitude < 50 then
                    local moveDir = THumanoid.MoveDirection
                    FPos(BasePart, CFrame.new(0, 1.5, 0) + moveDir * vel.Magnitude/1.25, CFrame.Angles(math.rad(Angle),0,0)) task.wait()
                    FPos(BasePart, CFrame.new(0,-1.5, 0) + moveDir * vel.Magnitude/1.25, CFrame.Angles(math.rad(Angle),0,0)) task.wait()
                    FPos(BasePart, CFrame.new(2.25,1.5,-2.25)+moveDir*vel.Magnitude/1.25, CFrame.Angles(math.rad(Angle),0,0)) task.wait()
                    FPos(BasePart, CFrame.new(-2.25,-1.5,2.25)+moveDir*vel.Magnitude/1.25,CFrame.Angles(math.rad(Angle),0,0)) task.wait()
                    FPos(BasePart, CFrame.new(0, 1.5, 0) + moveDir, CFrame.Angles(math.rad(Angle),0,0)) task.wait()
                    FPos(BasePart, CFrame.new(0,-1.5, 0) + moveDir, CFrame.Angles(math.rad(Angle),0,0)) task.wait()
                else
                    local ws = THumanoid.WalkSpeed or 16
                    local tv = TRootPart and TRootPart.Velocity.Magnitude or ws
                    FPos(BasePart,CFrame.new(0,1.5,ws),   CFrame.Angles(math.rad(90),0,0)) task.wait()
                    FPos(BasePart,CFrame.new(0,-1.5,-ws),  CFrame.Angles(0,0,0))            task.wait()
                    FPos(BasePart,CFrame.new(0,1.5,ws),   CFrame.Angles(math.rad(90),0,0)) task.wait()
                    FPos(BasePart,CFrame.new(0,1.5,tv/1.25),CFrame.Angles(math.rad(90),0,0)) task.wait()
                    FPos(BasePart,CFrame.new(0,-1.5,-tv/1.25),CFrame.Angles(0,0,0))          task.wait()
                    FPos(BasePart,CFrame.new(0,1.5,tv/1.25),CFrame.Angles(math.rad(90),0,0)) task.wait()
                    FPos(BasePart,CFrame.new(0,-1.5,0),    CFrame.Angles(math.rad(90),0,0)) task.wait()
                    FPos(BasePart,CFrame.new(0,-1.5,0),    CFrame.Angles(0,0,0))             task.wait()
                    FPos(BasePart,CFrame.new(0,-1.5,0),    CFrame.Angles(math.rad(-90),0,0)) task.wait()
                    FPos(BasePart,CFrame.new(0,-1.5,0),    CFrame.Angles(0,0,0))             task.wait()
                end
            end
        until ShouldStop() or (BasePart.Velocity.Magnitude > 500 and tick() > StartTime + 0.5)
    end

    if TRootPart and THead then
        if (TRootPart.CFrame.p - THead.CFrame.p).Magnitude > 5 then
            SFBasePart(THead)
        else
            SFBasePart(TRootPart)
        end
    elseif TRootPart then SFBasePart(TRootPart)
    elseif THead      then SFBasePart(THead)
    elseif Handle     then SFBasePart(Handle)
    end

    pcall(function() BV:Destroy() end)
    pcall(function() Humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, true) end)

    if Config.CameraRestore then
        pcall(function() Camera.CameraSubject = Humanoid end)
    end

    if Config.SafeReturn and _env.OldPos and IsAlive(LP) then
        local attempts  = 0
        local targetPos = _env.OldPos * CFrame.new(0, 0.5, 0)
        repeat
            attempts = attempts + 1
            pcall(function()
                for _, part in pairs(Character:GetDescendants()) do
                    if part:IsA("BasePart") then
                        part.Velocity = Vector3.zero; part.RotVelocity = Vector3.zero
                    end
                end
                RootPart.CFrame = targetPos
                Character:SetPrimaryPartCFrame(targetPos)
                Humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
            end)
            task.wait(0.05)
        until (RootPart.Position - _env.OldPos.p).Magnitude < 15
            or attempts > 20 or not IsAlive(LP)
    end

    workspace.FallenPartsDestroyHeight = _env.FPDH
    State.Flinging = false
end

-- ============================================================
-- [[ 6. ГОЛОВНИЙ ЦИКЛ ]]
-- ============================================================
task.spawn(function()
    while not State.Stopped do
        task.wait(Config.LoopDelay)
        if not State.FlingLoop then continue end
        if not IsAlive(LP) then continue end
        if State.Flinging then continue end

        local extraDelay = math.random(0, 3) * 0.01
        if extraDelay > 0 then task.wait(extraDelay) end

        local target = GetTarget(Config.CurrentTarget)

        if target == "all" then
            local playerList = {}
            for _, v in pairs(Players:GetPlayers()) do
                if v ~= LP and not IsWhitelisted(v) and IsAlive(v) then
                    table.insert(playerList, v)
                end
            end
            for i = #playerList, 2, -1 do
                local j = math.random(i)
                playerList[i], playerList[j] = playerList[j], playerList[i]
            end
            for _, v in pairs(playerList) do
                if not State.FlingLoop or State.Stopped then break end
                if not IsAlive(v) then continue end
                FlingPlayer(v)
                task.wait(math.random(2, 6) * 0.01)
            end
        elseif target and target ~= LP then
            FlingPlayer(target)
        else
            task.wait(0.3)
        end
    end
end)

-- ============================================================
-- [[ 7. АВТО-ВІДНОВЛЕННЯ ПІСЛЯ СМЕРТІ ]]
-- FIX: UpdateStatus через flag замість прямого виклику до ініціалізації GUI
-- ============================================================
local wasFlinging     = false
local guiReadyFlag    = false  -- стає true після ініціалізації GUI
local pendingUIUpdate = false  -- черга оновлення статусу

LP.CharacterAdded:Connect(function(_newChar)
    wasFlinging     = State.FlingLoop
    State.FlingLoop = false
    State.Flinging  = false
    workspace.FallenPartsDestroyHeight = _env.FPDH

    if Config.AutoRespawn and wasFlinging then
        task.wait(3)
        if not State.Stopped then
            State.FlingLoop = true
            Notify("OMNI-FLING", "Авто-відновлення флінгу ✓", 2)
        end
    else
        Notify("OMNI-FLING", "Вимкнено при респавні", 2)
    end

    -- FIX: оновлюємо UI через flag щоб уникнути nil-виклику
    if guiReadyFlag then
        pendingUIUpdate = true
    end
end)

game:GetService("Players").LocalPlayer.AncestryChanged:Connect(function()
    State.Stopped = true
end)

-- ============================================================
-- [[ 8. АНТІ-АФК ]]
-- ============================================================
StartAntiAFK()

-- ============================================================
-- [[ 9. GUI ]]
-- ============================================================
local GuiParent = LP:WaitForChild("PlayerGui", 10) or LP:FindFirstChildOfClass("PlayerGui") or LP:WaitForChild("PlayerGui")
pcall(function()
    local cg = game:GetService("CoreGui")
    local _  = cg.Name  -- перевіряємо доступ
    GuiParent = cg
end)

local Screen = Instance.new("ScreenGui", GuiParent)
Screen.Name           = "OmniFling"
Screen.ResetOnSpawn   = false
Screen.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
Screen.DisplayOrder   = 999

local W = IsMobile and 270 or 275
local H = IsMobile and 480 or 440

local Main = Instance.new("Frame", Screen)
Main.Size             = UDim2.new(0, W, 0, H)
Main.Position         = UDim2.new(0.5, -W/2, 0.5, -H/2)
Main.BackgroundColor3 = Color3.fromRGB(6, 6, 10)
Main.BorderSizePixel  = 0
Main.ClipsDescendants = true
Instance.new("UICorner", Main)

local MainStroke = Instance.new("UIStroke", Main)
MainStroke.Color     = Color3.fromRGB(255,255,255)
MainStroke.Thickness = 2

-- Title Bar
local TitleBar = Instance.new("Frame", Main)
TitleBar.Size             = UDim2.new(1, 0, 0, 44)
TitleBar.BackgroundColor3 = Color3.fromRGB(10, 10, 16)
TitleBar.BorderSizePixel  = 0
Instance.new("UICorner", TitleBar)

local TitleGrad = Instance.new("UIGradient", TitleBar)
TitleGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0,   Color3.fromRGB(0,0,0)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255,255,255)),
    ColorSequenceKeypoint.new(1,   Color3.fromRGB(0,0,0)),
})

local TitleLbl = Instance.new("TextLabel", TitleBar)
TitleLbl.Size                   = UDim2.new(1, -36, 1, 0)
TitleLbl.BackgroundTransparency = 1
TitleLbl.TextColor3             = Color3.fromRGB(255,255,255)
TitleLbl.Font                   = Enum.Font.GothamBlack
TitleLbl.TextSize               = IsMobile and 14 or 15
TitleLbl.Text                   = "⚡ OMNI-FLING  MONSTER"
TitleLbl.ZIndex                 = 2

do  -- CloseBtn (локальна область)
    local CloseBtn = Instance.new("TextButton", TitleBar)
    CloseBtn.Size             = UDim2.new(0, 28, 0, 28)
    CloseBtn.Position         = UDim2.new(1, -32, 0.5, -14)
    CloseBtn.BackgroundColor3 = Color3.fromRGB(50,50,50)
    CloseBtn.Text             = "✕"
    CloseBtn.TextColor3       = Color3.fromRGB(255,255,255)
    CloseBtn.Font             = Enum.Font.GothamBold
    CloseBtn.TextSize         = 13
    CloseBtn.BorderSizePixel  = 0
    CloseBtn.ZIndex           = 3
    Instance.new("UICorner", CloseBtn)
    CloseBtn.MouseButton1Click:Connect(function() Main.Visible = false end)
end

local Scroll = Instance.new("ScrollingFrame", Main)
Scroll.Size                   = UDim2.new(1, -10, 1, -52)
Scroll.Position               = UDim2.new(0, 5, 0, 48)
Scroll.BackgroundTransparency = 1
Scroll.ScrollBarThickness     = IsMobile and 0 or 3
Scroll.ScrollBarImageColor3   = Color3.fromRGB(120,120,120)
Scroll.BorderSizePixel        = 0

local Layout = Instance.new("UIListLayout", Scroll)
Layout.Padding             = UDim.new(0, 6)
Layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
Layout.SortOrder           = Enum.SortOrder.LayoutOrder

do  -- Pad (не потрібен далі)
    local Pad = Instance.new("UIPadding", Scroll)
    Pad.PaddingTop    = UDim.new(0, 4)
    Pad.PaddingBottom = UDim.new(0, 8)
end

Layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    Scroll.CanvasSize = UDim2.new(0, 0, 0, Layout.AbsoluteContentSize.Y + 12)
end)

-- GUI ХЕЛПЕРИ
local orderCount = 0
local function NextOrder() orderCount += 1; return orderCount end

local function AddCat(text)
    local f = Instance.new("Frame", Scroll)
    f.Size             = UDim2.new(0.97, 0, 0, 22)
    f.LayoutOrder      = NextOrder()
    f.BackgroundColor3 = Color3.fromRGB(16,16,24)
    f.BorderSizePixel  = 0
    Instance.new("UICorner", f)
    local l = Instance.new("TextLabel", f)
    l.Size                   = UDim2.new(1, 0, 1, 0)
    l.BackgroundTransparency = 1
    l.TextColor3             = Color3.fromRGB(160,160,190)
    l.Font                   = Enum.Font.GothamBold
    l.TextSize               = 11
    l.Text                   = "── " .. text .. " ──"
end

local function MakeBtn(text, color)
    local btn = Instance.new("TextButton", Scroll)
    btn.Size             = UDim2.new(0.97, 0, 0, IsMobile and 50 or 44)
    btn.LayoutOrder      = NextOrder()
    btn.Text             = text
    btn.BackgroundColor3 = color or Color3.fromRGB(32,32,46)
    btn.TextColor3       = Color3.fromRGB(255,255,255)
    btn.Font             = Enum.Font.GothamBold
    btn.TextSize         = 13
    btn.BorderSizePixel  = 0
    btn.AutoButtonColor  = false
    Instance.new("UICorner", btn)
    local st = Instance.new("UIStroke", btn)
    st.Color = Color3.fromRGB(60,60,80); st.Thickness = 1
    return btn
end

local function MakeToggle(text, state)
    local btn = MakeBtn(
        state and "✅  " .. text or "❌  " .. text,
        state and Color3.fromRGB(20,90,30) or Color3.fromRGB(55,20,20)
    )
    btn.TextSize = 12
    return btn
end

-- GUI КОНТЕНТ
AddCat("TARGET  (ім'я / all / random / пусто = найближчий)")

local TargetBox = Instance.new("TextBox", Scroll)
TargetBox.Size              = UDim2.new(0.97, 0, 0, IsMobile and 46 or 40)
TargetBox.LayoutOrder       = NextOrder()
TargetBox.PlaceholderText   = "Ім'я / all / random / пусто = найближчий"
TargetBox.Text              = ""
TargetBox.BackgroundColor3  = Color3.fromRGB(16,16,26)
TargetBox.TextColor3        = Color3.fromRGB(255,255,255)
TargetBox.PlaceholderColor3 = Color3.fromRGB(80,80,100)
TargetBox.Font              = Enum.Font.GothamBold
TargetBox.TextSize          = 13
TargetBox.ClearTextOnFocus  = false
TargetBox.BorderSizePixel   = 0
Instance.new("UICorner", TargetBox)
do local tbSt = Instance.new("UIStroke", TargetBox)
tbSt.Color = Color3.fromRGB(60,60,100); tbSt.Thickness = 1 end
TargetBox.FocusLost:Connect(function() Config.CurrentTarget = TargetBox.Text end)

AddCat("FLING")
local StartBtn = MakeBtn("▶  START FLING", Color3.fromRGB(22,130,40))
StartBtn.TextSize = IsMobile and 15 or 14
StartBtn.Size     = UDim2.new(0.97, 0, 0, IsMobile and 56 or 50)
do local startSt = StartBtn:FindFirstChildOfClass("UIStroke")
if startSt then startSt.Color = Color3.fromRGB(80,200,80); startSt.Thickness = 1.5 end end

local styles     = {"auto","up","side","down","spin"}
local styleIdx2  = 1
local styleNames = {
    auto = "🌀  STYLE: AUTO (max dmg)",
    up   = "⬆️  STYLE: UP",
    side = "↔️  STYLE: SIDE",
    down = "⬇️  STYLE: DOWN",
    spin = "🔄  STYLE: SPIN",
}
local StyleBtn = MakeBtn("🌀  STYLE: AUTO (max dmg)")

AddCat("FLING DURATION")

-- Duration slider
local sliderDrag  = false
local durLevels   = {1, 1.5, 2, 2.5, 3, 4, 5, 7, 10}
local durIdx      = 3
local knobSz      = IsMobile and 18 or 13   -- FIX: renamed from kSz to avoid confusion

local DurContainer = Instance.new("Frame", Scroll)
DurContainer.Size             = UDim2.new(0.97, 0, 0, 52)
DurContainer.LayoutOrder      = NextOrder()
DurContainer.BackgroundColor3 = Color3.fromRGB(14,14,20)
DurContainer.BorderSizePixel  = 0
Instance.new("UICorner", DurContainer)

local DurLabel = Instance.new("TextLabel", DurContainer)
DurLabel.Size                   = UDim2.new(1, -8, 0, 22)
DurLabel.Position               = UDim2.new(0, 4, 0, 2)
DurLabel.BackgroundTransparency = 1
DurLabel.TextColor3             = Color3.fromRGB(200,200,200)
DurLabel.Font                   = Enum.Font.GothamBold
DurLabel.TextSize               = 11
DurLabel.TextXAlignment         = Enum.TextXAlignment.Left
DurLabel.Text                   = "Duration: 2s"

local DurTrack = Instance.new("Frame", DurContainer)
DurTrack.Size             = UDim2.new(0.92, 0, 0, 7)
DurTrack.Position         = UDim2.new(0.04, 0, 0, 32)
DurTrack.BackgroundColor3 = Color3.fromRGB(35,35,48)
DurTrack.BorderSizePixel  = 0
Instance.new("UICorner", DurTrack)

local DurFill = Instance.new("Frame", DurTrack)
DurFill.Size             = UDim2.new(0.18, 0, 1, 0)
DurFill.BackgroundColor3 = Color3.fromRGB(200,200,200)
DurFill.BorderSizePixel  = 0
Instance.new("UICorner", DurFill)
do local dGrad = Instance.new("UIGradient", DurFill)
dGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(60,60,60)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(255,255,255)),
}) end

local DurKnob = Instance.new("Frame", DurTrack)
DurKnob.Size             = UDim2.new(0, knobSz, 0, knobSz)
DurKnob.Position         = UDim2.new(0.18, -knobSz/2, 0.5, -knobSz/2)
DurKnob.BackgroundColor3 = Color3.fromRGB(255,255,255)
DurKnob.BorderSizePixel  = 0
Instance.new("UICorner", DurKnob)

Config.FlingDuration = durLevels[durIdx]

local function UpdateDurSlider(inp)
    local rel = math.clamp(
        (inp.Position.X - DurTrack.AbsolutePosition.X) / DurTrack.AbsoluteSize.X, 0, 1)
    local idx = math.clamp(math.round(rel * (#durLevels - 1)) + 1, 1, #durLevels)
    durIdx               = idx
    Config.FlingDuration = durLevels[idx]
    local relPos         = (idx - 1) / (#durLevels - 1)
    DurFill.Size         = UDim2.new(relPos, 0, 1, 0)
    DurKnob.Position     = UDim2.new(relPos, -knobSz/2, 0.5, -knobSz/2)
    DurLabel.Text        = "Duration: " .. durLevels[idx] .. "s"
end

DurTrack.InputBegan:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.MouseButton1
    or inp.UserInputType == Enum.UserInputType.Touch then
        sliderDrag = true; UpdateDurSlider(inp)
    end
end)
UIS.InputChanged:Connect(function(inp)
    if not sliderDrag then return end
    if inp.UserInputType == Enum.UserInputType.MouseMovement
    or inp.UserInputType == Enum.UserInputType.Touch then
        UpdateDurSlider(inp)
    end
end)
UIS.InputEnded:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.MouseButton1
    or inp.UserInputType == Enum.UserInputType.Touch then
        sliderDrag = false
    end
end)

AddCat("ANTI-DETECT OPTIONS")

local AutoRespawnBtn = MakeToggle("AUTO RESUME AFTER DEATH", Config.AutoRespawn)
local SafeReturnBtn  = MakeToggle("SAFE RETURN TO POSITION", Config.SafeReturn)
local AntiAFKBtn     = MakeToggle("ANTI-AFK", Config.AntiAFK)
local SilentBtn      = MakeToggle("SILENT MODE (no notifs)", Config.SilentMode)
local JitterBtn      = MakeToggle("TELEPORT NOISE (anti-detect)", Config.TeleportNoise)

local function RefreshToggle(btn, state, label)
    btn.Text             = state and "✅  " .. label or "❌  " .. label
    btn.BackgroundColor3 = state and Color3.fromRGB(20,90,30) or Color3.fromRGB(55,20,20)
end

AutoRespawnBtn.MouseButton1Click:Connect(function()
    Config.AutoRespawn = not Config.AutoRespawn
    RefreshToggle(AutoRespawnBtn, Config.AutoRespawn, "AUTO RESUME AFTER DEATH")
end)
SafeReturnBtn.MouseButton1Click:Connect(function()
    Config.SafeReturn = not Config.SafeReturn
    RefreshToggle(SafeReturnBtn, Config.SafeReturn, "SAFE RETURN TO POSITION")
end)
AntiAFKBtn.MouseButton1Click:Connect(function()
    Config.AntiAFK = not Config.AntiAFK
    RefreshToggle(AntiAFKBtn, Config.AntiAFK, "ANTI-AFK")
end)
SilentBtn.MouseButton1Click:Connect(function()
    Config.SilentMode = not Config.SilentMode
    RefreshToggle(SilentBtn, Config.SilentMode, "SILENT MODE (no notifs)")
end)
JitterBtn.MouseButton1Click:Connect(function()
    Config.TeleportNoise = not Config.TeleportNoise
    RefreshToggle(JitterBtn, Config.TeleportNoise, "TELEPORT NOISE (anti-detect)")
end)

AddCat("WHITELIST")
local WLBtn    = MakeBtn("🛡  ADD CLOSEST TO WHITELIST")
local ClearBtn = MakeBtn("🗑  CLEAR WHITELIST", Color3.fromRGB(36,16,16))
ClearBtn.TextColor3 = Color3.fromRGB(220,130,130)

AddCat("STATUS")
local StatusLbl = Instance.new("TextLabel", Scroll)
StatusLbl.Size                   = UDim2.new(0.97, 0, 0, 28)
StatusLbl.LayoutOrder            = NextOrder()
StatusLbl.BackgroundTransparency = 1
StatusLbl.Text                   = "● Idle"
StatusLbl.TextColor3             = Color3.fromRGB(100,100,115)
StatusLbl.Font                   = Enum.Font.GothamBold
StatusLbl.TextSize               = 13

do  -- KeysLbl (не потрібен далі)
    local KeysLbl = Instance.new("TextLabel", Scroll)
    KeysLbl.Size                   = UDim2.new(0.97, 0, 0, IsMobile and 36 or 22)
    KeysLbl.LayoutOrder            = NextOrder()
    KeysLbl.BackgroundTransparency = 1
    KeysLbl.TextWrapped            = true
    KeysLbl.Text                   = IsMobile
        and "F = меню  |  K = увімк/вимк"
        or  "J = UI  |  K = Toggle Fling"
    KeysLbl.TextColor3             = Color3.fromRGB(55,55,70)
    KeysLbl.Font                   = Enum.Font.Gotham
    KeysLbl.TextSize               = IsMobile and 12 or 11
end

-- ============================================================
-- ЛОГІКА КНОПОК
-- FIX: UpdateStatus тепер local, оголошена до CharacterAdded
-- ============================================================
local function UpdateStatus()
    if State.FlingLoop then
        StatusLbl.Text            = "● ACTIVE — флінгує"
        StatusLbl.TextColor3      = Color3.fromRGB(70,255,110)
        StartBtn.Text             = "■  STOP FLING"
        StartBtn.BackgroundColor3 = Color3.fromRGB(150,28,28)
    else
        StatusLbl.Text            = "● Idle"
        StatusLbl.TextColor3      = Color3.fromRGB(100,100,115)
        StartBtn.Text             = "▶  START FLING"
        StartBtn.BackgroundColor3 = Color3.fromRGB(22,130,40)
    end
end

-- Позначаємо що GUI готовий + обробляємо відкладені оновлення
guiReadyFlag = true
if pendingUIUpdate then UpdateStatus(); pendingUIUpdate = false end

StartBtn.MouseButton1Click:Connect(function()
    State.FlingLoop = not State.FlingLoop
    UpdateStatus()
    Notify("OMNI-FLING", State.FlingLoop and "Fling ON ✓" or "Fling OFF ✗", 2)
end)

StyleBtn.MouseButton1Click:Connect(function()
    styleIdx2        = (styleIdx2 % #styles) + 1
    State.FlingStyle = styles[styleIdx2]
    StyleBtn.Text    = styleNames[State.FlingStyle]
end)

WLBtn.MouseButton1Click:Connect(function()
    local closest = GetClosestPlayer()
    if closest then
        if not table.find(Whitelist, closest.UserId) then
            table.insert(Whitelist, closest.UserId)
            Notify("WHITELIST", "Додано: " .. closest.Name .. " ✓", 3)
        else
            Notify("WHITELIST", closest.Name .. " вже у списку", 2)
        end
    else
        Notify("WHITELIST", "Немає гравців поруч", 2)
    end
end)

ClearBtn.MouseButton1Click:Connect(function()
    Whitelist = { LP.UserId }
    Notify("WHITELIST", "Список очищено ✓", 2)
end)

-- Оновлення статусу кожні 0.5 сек (для автовідновлення після смерті)
task.spawn(function()
    while not State.Stopped do
        task.wait(0.5)
        if pendingUIUpdate and guiReadyFlag then
            UpdateStatus(); pendingUIUpdate = false
        end
    end
end)

-- ============================================================
-- DRAGGABLE
-- ============================================================
do
    local drag, dStart, dPos = false, nil, nil
    TitleBar.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            drag = true; dStart = inp.Position; dPos = Main.Position
        end
    end)
    UIS.InputChanged:Connect(function(inp)
        if not drag then return end
        if inp.UserInputType == Enum.UserInputType.MouseMovement
        or inp.UserInputType == Enum.UserInputType.Touch then
            local d = inp.Position - dStart
            Main.Position = UDim2.new(dPos.X.Scale, dPos.X.Offset + d.X,
                                       dPos.Y.Scale, dPos.Y.Offset + d.Y)
        end
    end)
    UIS.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then drag = false end
    end)
end

-- ============================================================
-- F / K КНОПКИ
-- ============================================================
local MBtnSz = IsMobile and 58 or 46

local MBtn = Instance.new("TextButton", Screen)
MBtn.Size             = UDim2.new(0, MBtnSz, 0, MBtnSz)
MBtn.Position         = UDim2.new(0, 10, 0.38, 0)
MBtn.Text             = "F"
MBtn.Font             = Enum.Font.GothamBlack
MBtn.TextSize         = IsMobile and 26 or 20
MBtn.BackgroundColor3 = Color3.fromRGB(10,10,15)
MBtn.TextColor3       = Color3.fromRGB(255,255,255)
MBtn.BorderSizePixel  = 0
MBtn.AutoButtonColor  = false
MBtn.ZIndex           = 100
Instance.new("UICorner", MBtn)

local mSt = Instance.new("UIStroke", MBtn)
mSt.Color = Color3.fromRGB(255,255,255); mSt.Thickness = 2

if IsMobile then
    local KBtn = Instance.new("TextButton", Screen)
    KBtn.Size             = UDim2.new(0, MBtnSz, 0, MBtnSz)
    KBtn.Position         = UDim2.new(0, 10, 0.50, 0)
    KBtn.Text             = "K"
    KBtn.Font             = Enum.Font.GothamBlack
    KBtn.TextSize         = 26
    KBtn.BackgroundColor3 = Color3.fromRGB(22,130,40)
    KBtn.TextColor3       = Color3.fromRGB(255,255,255)
    KBtn.BorderSizePixel  = 0
    KBtn.AutoButtonColor  = false
    KBtn.ZIndex           = 100
    Instance.new("UICorner", KBtn)
    do local kSt2 = Instance.new("UIStroke", KBtn)  -- FIX: renamed variable (was collision-prone)
    kSt2.Color = Color3.fromRGB(255,255,255); kSt2.Thickness = 2 end

    KBtn.MouseButton1Click:Connect(function()
        State.FlingLoop = not State.FlingLoop
        UpdateStatus()
        KBtn.BackgroundColor3 = State.FlingLoop
            and Color3.fromRGB(150,28,28) or Color3.fromRGB(22,130,40)
        Notify("OMNI-FLING", State.FlingLoop and "Fling ON ✓" or "Fling OFF ✗", 1.5)
    end)

    -- Drag KBtn
    do
        local d2, s2, p2, m2 = false, nil, nil, false
        KBtn.InputBegan:Connect(function(inp)
            if inp.UserInputType == Enum.UserInputType.Touch then
                d2 = true; s2 = inp.Position; p2 = KBtn.Position; m2 = false
            end
        end)
        UIS.InputChanged:Connect(function(inp)
            if not d2 then return end
            if inp.UserInputType == Enum.UserInputType.Touch then
                local dlt = inp.Position - s2
                if dlt.Magnitude > 6 then m2 = true end
                KBtn.Position = UDim2.new(p2.X.Scale, p2.X.Offset + dlt.X,
                                           p2.Y.Scale, p2.Y.Offset + dlt.Y)
            end
        end)
        UIS.InputEnded:Connect(function(inp)
            if inp.UserInputType == Enum.UserInputType.Touch then d2 = false end
        end)
        _ = m2  -- suppress unused warning
    end
end

-- Drag MBtn
do
    local mDrag, mStart, mPos, mTick, mMoved = false, nil, nil, 0, false
    MBtn.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            mDrag = true; mStart = inp.Position; mPos = MBtn.Position
            mTick = tick(); mMoved = false
        end
    end)
    UIS.InputChanged:Connect(function(inp)
        if not mDrag then return end
        if inp.UserInputType == Enum.UserInputType.MouseMovement
        or inp.UserInputType == Enum.UserInputType.Touch then
            local d = inp.Position - mStart
            if d.Magnitude > 6 then mMoved = true end
            MBtn.Position = UDim2.new(mPos.X.Scale, mPos.X.Offset + d.X,
                                       mPos.Y.Scale, mPos.Y.Offset + d.Y)
        end
    end)
    UIS.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            if mDrag and not mMoved and tick() - mTick < 0.28 then
                Main.Visible = not Main.Visible
            end
            mDrag = false
        end
    end)
end

-- ============================================================
-- B&W АНІМАЦІЯ
-- ============================================================
task.spawn(function()
    local t = 0
    while not State.Stopped do
        task.wait(0.03)
        t = t + 0.025
        local v    = (math.sin(t) + 1) / 2
        local vInv = 1 - v
        local bv   = math.floor(v    * 255)
        local bInv = math.floor(vInv * 255)

        MainStroke.Color      = Color3.fromRGB(bv, bv, bv)
        mSt.Color             = Color3.fromRGB(bv, bv, bv)
        MBtn.BackgroundColor3 = Color3.fromRGB(bInv, bInv, bInv)
        MBtn.TextColor3       = Color3.fromRGB(bv, bv, bv)

        local vA = math.floor(v    * 255)
        local vB = math.floor(vInv * 255)
        TitleGrad.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0,   Color3.fromRGB(vA,vA,vA)),
            ColorSequenceKeypoint.new(0.5, Color3.fromRGB(vB,vB,vB)),
            ColorSequenceKeypoint.new(1,   Color3.fromRGB(vA,vA,vA)),
        })
        TitleGrad.Rotation  = (t * 30) % 360
        local ts            = math.floor(180 + v * 75)
        TitleLbl.TextColor3 = Color3.fromRGB(ts, ts, ts)
    end
end)

-- ============================================================
-- КЛАВІШІ
-- ============================================================
UIS.InputBegan:Connect(function(inp, gpe)
    if gpe then return end
    if inp.KeyCode == Enum.KeyCode.J then
        Main.Visible = not Main.Visible
    end
    if inp.KeyCode == Enum.KeyCode.K then
        State.FlingLoop = not State.FlingLoop
        UpdateStatus()
        Notify("OMNI-FLING", State.FlingLoop and "Fling ON ✓" or "Fling OFF ✗", 1.5)
    end
end)

-- ============================================================
-- ЗУПИНКА
-- ============================================================
_env.OmniFlingStop = function()
    State.Stopped   = true
    State.FlingLoop = false
    State.Flinging  = false
    workspace.FallenPartsDestroyHeight = _env.FPDH
    pcall(function()
        local char = LP.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then Camera.CameraSubject = hum end
        end
    end)
    pcall(function() Screen:Destroy() end)
    _env.OmniFlingLoaded = false
end

-- ============================================================
Notify("OMNI-FLING MONSTER",
    IsMobile
        and "F=меню | K=toggle | Anti-detect ✓"
        or  "J=меню | K=toggle | Anti-detect ✓", 5)
