-- [[ V260.51: OMNI-FLING - ULTIMATE EDITION ]]
-- [[ Базується на SkidFling логіці + GUI + Mobile ]]

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS        = game:GetService("UserInputService")
local Workspace  = game:GetService("Workspace")
local StarterGui = game:GetService("StarterGui")
local TweenService = game:GetService("TweenService")

local LP     = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- ============================================================
-- [[ 0. ЗАХИСТ ВІД ПОДВІЙНОГО ЗАПУСКУ ]]
-- ============================================================
pcall(function()
    for _, sg in pairs({
        game:GetService("CoreGui"),
        LP:WaitForChild("PlayerGui")
    }) do
        for _, v in pairs(sg:GetChildren()) do
            if v:IsA("ScreenGui") and v.Name == "OmniFling" then
                v:Destroy()
            end
        end
    end
end)

-- ============================================================
-- [[ 1. КОНФІГ ]]
-- ============================================================
getgenv().FPDH = getgenv().FPDH or workspace.FallenPartsDestroyHeight

local Whitelist = { LP.UserId }

local State = {
    FlingLoop  = false,
    FlingStyle = "auto",  -- auto | up | side | down
    Flinging   = false,
}

local Config = {
    CurrentTarget = "",
    FlingDuration = 2,    -- секунди на один флінг
    LoopDelay     = 0.05, -- затримка між циклами
}

local IsMobile = UIS.TouchEnabled

-- ============================================================
-- [[ 2. УТИЛІТИ ]]
-- ============================================================
local function Notify(title, text, dur)
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
        if n:match("^" .. input) or dn:match("^" .. input) then
            return v
        end
    end
    return nil
end

-- ============================================================
-- [[ 3. CORE FLING ЛОГІКА (SkidFling метод — найсильніший) ]]
-- ============================================================
local function FlingPlayer(TargetPlayer)
    if not TargetPlayer then return end
    if IsWhitelisted(TargetPlayer) then return end
    if not IsAlive(TargetPlayer) then return end
    if State.Flinging then return end
    if not IsAlive(LP) then return end

    State.Flinging = true

    -- Наш персонаж
    local Character = LP.Character
    local Humanoid  = Character and Character:FindFirstChildOfClass("Humanoid")
    local RootPart  = Humanoid  and Humanoid.RootPart

    if not Character or not Humanoid or not RootPart then
        State.Flinging = false
        return
    end

    -- Ціль
    local TCharacter = TargetPlayer.Character
    local THumanoid  = TCharacter and TCharacter:FindFirstChildOfClass("Humanoid")
    local TRootPart  = THumanoid  and THumanoid.RootPart
    local THead      = TCharacter and TCharacter:FindFirstChild("Head")
    local Accessory  = TCharacter and TCharacter:FindFirstChildOfClass("Accessory")
    local Handle     = Accessory  and Accessory:FindFirstChild("Handle")

    if not THumanoid then
        State.Flinging = false
        return
    end

    -- Зберігаємо позицію
    if RootPart.Velocity.Magnitude < 50 then
        getgenv().OldPos = RootPart.CFrame
    end

    -- Камера на ціль
    if THead then
        Camera.CameraSubject = THead
    elseif Handle then
        Camera.CameraSubject = Handle
    elseif THumanoid then
        Camera.CameraSubject = THumanoid
    end

    -- Розблоковуємо висоту падіння
    workspace.FallenPartsDestroyHeight = 0/0

    -- BodyVelocity на нас
    local BV = Instance.new("BodyVelocity")
    BV.Name      = "OmniFlingVel"
    BV.Parent    = RootPart
    BV.Velocity  = Vector3.new(9e8, 9e8, 9e8)
    BV.MaxForce  = Vector3.new(1/0, 1/0, 1/0)

    Humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, false)
    pcall(function() Humanoid.Sit = false end)

    -- Функція позиціонування
    local function FPos(BasePart, Pos, Ang)
        local newCF = CFrame.new(BasePart.Position) * Pos * Ang
        pcall(function()
            RootPart.CFrame = newCF
            Character:SetPrimaryPartCFrame(newCF)
            RootPart.Velocity    = Vector3.new(9e7, 9e7 * 10, 9e7)
            RootPart.RotVelocity = Vector3.new(9e8, 9e8, 9e8)
        end)
    end

    -- Основний цикл флінгу (адаптовано під стиль)
    local function SFBasePart(BasePart)
        local TimeToWait = Config.FlingDuration
        local Time       = tick()
        local Angle      = 0

        -- Кастомний стиль
        local function GetStyleOffset()
            if State.FlingStyle == "up" then
                return CFrame.new(0, 2, 0), CFrame.Angles(math.rad(Angle), 0, 0)
            elseif State.FlingStyle == "side" then
                return CFrame.new(2, 0, 0), CFrame.Angles(0, math.rad(Angle), 0)
            elseif State.FlingStyle == "down" then
                return CFrame.new(0, -2, 0), CFrame.Angles(math.rad(Angle), 0, 0)
            else
                -- auto (найефективніший — оригінальний SkidFling)
                return nil, nil
            end
        end

        repeat
            if not RootPart or not THumanoid then break end
            if not State.FlingLoop then break end

            Angle = Angle + 100

            local stylePos, styleAng = GetStyleOffset()

            if stylePos then
                -- Кастомний стиль
                FPos(BasePart, stylePos, styleAng)
                task.wait()
                FPos(BasePart, CFrame.new(0, -stylePos.Y, 0), styleAng)
                task.wait()
            else
                -- AUTO (оригінальний SkidFling — найсильніший)
                if BasePart.Velocity.Magnitude < 50 then
                    FPos(BasePart,
                        CFrame.new(0, 1.5, 0) + THumanoid.MoveDirection * BasePart.Velocity.Magnitude / 1.25,
                        CFrame.Angles(math.rad(Angle), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, -1.5, 0) + THumanoid.MoveDirection * BasePart.Velocity.Magnitude / 1.25,
                        CFrame.Angles(math.rad(Angle), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(2.25, 1.5, -2.25) + THumanoid.MoveDirection * BasePart.Velocity.Magnitude / 1.25,
                        CFrame.Angles(math.rad(Angle), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(-2.25, -1.5, 2.25) + THumanoid.MoveDirection * BasePart.Velocity.Magnitude / 1.25,
                        CFrame.Angles(math.rad(Angle), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, 1.5, 0) + THumanoid.MoveDirection,
                        CFrame.Angles(math.rad(Angle), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, -1.5, 0) + THumanoid.MoveDirection,
                        CFrame.Angles(math.rad(Angle), 0, 0))
                    task.wait()
                else
                    -- Якщо ціль біжить — адаптуємось до її швидкості
                    FPos(BasePart,
                        CFrame.new(0, 1.5, THumanoid.WalkSpeed),
                        CFrame.Angles(math.rad(90), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, -1.5, -THumanoid.WalkSpeed),
                        CFrame.Angles(0, 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, 1.5, THumanoid.WalkSpeed),
                        CFrame.Angles(math.rad(90), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, 1.5, TRootPart and TRootPart.Velocity.Magnitude / 1.25 or 10),
                        CFrame.Angles(math.rad(90), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, -1.5, -(TRootPart and TRootPart.Velocity.Magnitude / 1.25 or 10)),
                        CFrame.Angles(0, 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, 1.5, TRootPart and TRootPart.Velocity.Magnitude / 1.25 or 10),
                        CFrame.Angles(math.rad(90), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, -1.5, 0),
                        CFrame.Angles(math.rad(90), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, -1.5, 0),
                        CFrame.Angles(0, 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, -1.5, 0),
                        CFrame.Angles(math.rad(-90), 0, 0))
                    task.wait()

                    FPos(BasePart,
                        CFrame.new(0, -1.5, 0),
                        CFrame.Angles(0, 0, 0))
                    task.wait()
                end
            end

        until BasePart.Velocity.Magnitude > 500
            or BasePart.Parent ~= TargetPlayer.Character
            or TargetPlayer.Parent ~= Players
            or THumanoid.Sit
            or Humanoid.Health <= 0
            or tick() > Time + TimeToWait
            or not State.FlingLoop
    end

    -- Вибір BasePart
    if TRootPart and THead then
        if (TRootPart.CFrame.p - THead.CFrame.p).Magnitude > 5 then
            SFBasePart(THead)
        else
            SFBasePart(TRootPart)
        end
    elseif TRootPart then
        SFBasePart(TRootPart)
    elseif THead then
        SFBasePart(THead)
    elseif Handle then
        SFBasePart(Handle)
    end

    -- Cleanup
    pcall(function() BV:Destroy() end)
    Humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, true)

    -- Відновлення камери
    pcall(function()
        Camera.CameraSubject = Humanoid
    end)

    -- Телепорт назад
    if getgenv().OldPos then
        local attempts = 0
        repeat
            attempts = attempts + 1
            pcall(function()
                RootPart.CFrame = getgenv().OldPos * CFrame.new(0, 0.5, 0)
                Character:SetPrimaryPartCFrame(getgenv().OldPos * CFrame.new(0, 0.5, 0))
                Humanoid:ChangeState("GettingUp")
                for _, x in pairs(Character:GetChildren()) do
                    if x:IsA("BasePart") then
                        x.Velocity    = Vector3.zero
                        x.RotVelocity = Vector3.zero
                    end
                end
            end)
            task.wait()
        until (RootPart.Position - getgenv().OldPos.p).Magnitude < 25
            or attempts > 30
    end

    workspace.FallenPartsDestroyHeight = getgenv().FPDH
    State.Flinging = false
end

-- ============================================================
-- [[ 4. ГОЛОВНИЙ ЦИКЛ ]]
-- ============================================================
task.spawn(function()
    while true do
        task.wait(Config.LoopDelay)

        if not State.FlingLoop then continue end
        if not IsAlive(LP) then continue end
        if State.Flinging then continue end

        local target = GetTarget(Config.CurrentTarget)

        if target == "all" then
            for _, v in pairs(Players:GetPlayers()) do
                if not State.FlingLoop then break end
                if v == LP or IsWhitelisted(v) then continue end
                if not IsAlive(v) then continue end
                FlingPlayer(v)
            end
        elseif target and target ~= LP then
            FlingPlayer(target)
        else
            task.wait(0.3)
        end
    end
end)

LP.CharacterAdded:Connect(function()
    State.FlingLoop  = false
    State.Flinging   = false
    workspace.FallenPartsDestroyHeight = getgenv().FPDH
    Notify("OMNI-FLING", "Вимкнено при респавні", 2)
end)

-- ============================================================
-- [[ 5. GUI ]]
-- ============================================================
local GuiParent = LP:WaitForChild("PlayerGui")
pcall(function()
    local cg = game:GetService("CoreGui")
    local _  = cg.Name
    GuiParent = cg
end)

local Screen = Instance.new("ScreenGui", GuiParent)
Screen.Name           = "OmniFling"
Screen.ResetOnSpawn   = false
Screen.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local W = IsMobile and 265 or 270
local H = IsMobile and 430 or 410

local Main = Instance.new("Frame", Screen)
Main.Size             = UDim2.new(0, W, 0, H)
Main.Position         = UDim2.new(0.5, -W/2, 0.5, -H/2)
Main.BackgroundColor3 = Color3.fromRGB(8, 8, 12)
Main.BorderSizePixel  = 0
Instance.new("UICorner", Main)

local MainStroke = Instance.new("UIStroke", Main)
MainStroke.Color     = Color3.fromRGB(255, 255, 255)
MainStroke.Thickness = 2

-- Title Bar
local TitleBar = Instance.new("Frame", Main)
TitleBar.Size             = UDim2.new(1, 0, 0, 42)
TitleBar.BackgroundColor3 = Color3.fromRGB(12, 12, 18)
TitleBar.BorderSizePixel  = 0
Instance.new("UICorner", TitleBar)

local TitleGrad = Instance.new("UIGradient", TitleBar)
TitleGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0,   Color3.fromRGB(0,   0,   0)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 255, 255)),
    ColorSequenceKeypoint.new(1,   Color3.fromRGB(0,   0,   0)),
})

local TitleLbl = Instance.new("TextLabel", TitleBar)
TitleLbl.Size                   = UDim2.new(1, 0, 1, 0)
TitleLbl.BackgroundTransparency = 1
TitleLbl.TextColor3             = Color3.fromRGB(255, 255, 255)
TitleLbl.Font                   = Enum.Font.GothamBlack
TitleLbl.TextSize               = IsMobile and 14 or 15
TitleLbl.Text                   = "⚡ OMNI-FLING V260.51"
TitleLbl.ZIndex                 = 2

local CloseBtn = Instance.new("TextButton", TitleBar)
CloseBtn.Size             = UDim2.new(0, 28, 0, 28)
CloseBtn.Position         = UDim2.new(1, -32, 0.5, -14)
CloseBtn.BackgroundColor3 = Color3.fromRGB(55, 55, 55)
CloseBtn.Text             = "✕"
CloseBtn.TextColor3       = Color3.fromRGB(255, 255, 255)
CloseBtn.Font             = Enum.Font.GothamBold
CloseBtn.TextSize         = 13
CloseBtn.BorderSizePixel  = 0
CloseBtn.ZIndex           = 3
Instance.new("UICorner", CloseBtn)
CloseBtn.MouseButton1Click:Connect(function()
    Main.Visible = false
end)

-- Scroll
local Scroll = Instance.new("ScrollingFrame", Main)
Scroll.Size                   = UDim2.new(1, -10, 1, -50)
Scroll.Position               = UDim2.new(0, 5, 0, 46)
Scroll.BackgroundTransparency = 1
Scroll.ScrollBarThickness     = IsMobile and 0 or 3
Scroll.ScrollBarImageColor3   = Color3.fromRGB(120, 120, 120)
Scroll.BorderSizePixel        = 0

local Layout = Instance.new("UIListLayout", Scroll)
Layout.Padding             = UDim.new(0, 6)
Layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
Layout.SortOrder           = Enum.SortOrder.LayoutOrder

local Pad = Instance.new("UIPadding", Scroll)
Pad.PaddingTop    = UDim.new(0, 4)
Pad.PaddingBottom = UDim.new(0, 8)

Layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    Scroll.CanvasSize = UDim2.new(0, 0, 0, Layout.AbsoluteContentSize.Y + 12)
end)

-- ============================================================
-- [[ GUI ХЕЛПЕРИ ]]
-- ============================================================
local orderCount = 0
local function NextOrder()
    orderCount = orderCount + 1
    return orderCount
end

local function AddCat(text)
    local f = Instance.new("Frame", Scroll)
    f.Size             = UDim2.new(0.97, 0, 0, 22)
    f.LayoutOrder      = NextOrder()
    f.BackgroundColor3 = Color3.fromRGB(18, 18, 26)
    f.BorderSizePixel  = 0
    Instance.new("UICorner", f)
    local l = Instance.new("TextLabel", f)
    l.Size                   = UDim2.new(1, 0, 1, 0)
    l.BackgroundTransparency = 1
    l.TextColor3             = Color3.fromRGB(180, 180, 200)
    l.Font                   = Enum.Font.GothamBold
    l.TextSize               = 11
    l.Text                   = "── " .. text .. " ──"
end

local function MakeBtn(text, color)
    local btn = Instance.new("TextButton", Scroll)
    btn.Size             = UDim2.new(0.97, 0, 0, IsMobile and 50 or 44)
    btn.LayoutOrder      = NextOrder()
    btn.Text             = text
    btn.BackgroundColor3 = color or Color3.fromRGB(35, 35, 48)
    btn.TextColor3       = Color3.fromRGB(255, 255, 255)
    btn.Font             = Enum.Font.GothamBold
    btn.TextSize         = IsMobile and 13 or 13
    btn.BorderSizePixel  = 0
    btn.AutoButtonColor  = false
    Instance.new("UICorner", btn)
    local st = Instance.new("UIStroke", btn)
    st.Color     = Color3.fromRGB(70, 70, 90)
    st.Thickness = 1
    return btn
end

-- ============================================================
-- [[ GUI КОНТЕНТ ]]
-- ============================================================

-- TARGET
AddCat("TARGET  (ім'я / all / random / пусто = найближчий)")

local TargetBox = Instance.new("TextBox", Scroll)
TargetBox.Size              = UDim2.new(0.97, 0, 0, IsMobile and 46 or 40)
TargetBox.LayoutOrder       = NextOrder()
TargetBox.PlaceholderText   = "Ім'я / all / random / пусто = найближчий"
TargetBox.Text              = ""
TargetBox.BackgroundColor3  = Color3.fromRGB(18, 18, 28)
TargetBox.TextColor3        = Color3.fromRGB(255, 255, 255)
TargetBox.PlaceholderColor3 = Color3.fromRGB(90, 90, 105)
TargetBox.Font              = Enum.Font.GothamBold
TargetBox.TextSize          = IsMobile and 13 or 13
TargetBox.ClearTextOnFocus  = false
TargetBox.BorderSizePixel   = 0
Instance.new("UICorner", TargetBox)
local tbSt = Instance.new("UIStroke", TargetBox)
tbSt.Color = Color3.fromRGB(70, 70, 100); tbSt.Thickness = 1

TargetBox.FocusLost:Connect(function()
    Config.CurrentTarget = TargetBox.Text
end)

-- FLING
AddCat("FLING")

local StartBtn = MakeBtn("▶  START FLING", Color3.fromRGB(22, 130, 40))
StartBtn.TextSize = IsMobile and 15 or 14
StartBtn.Size     = UDim2.new(0.97, 0, 0, IsMobile and 56 or 50)
local startSt = StartBtn:FindFirstChildOfClass("UIStroke")
if startSt then startSt.Color = Color3.fromRGB(100, 200, 100); startSt.Thickness = 1.5 end

local StyleBtn = MakeBtn("🌀  STYLE: AUTO")

-- DURATION слайдер
AddCat("FLING DURATION")

local DurContainer = Instance.new("Frame", Scroll)
DurContainer.Size             = UDim2.new(0.97, 0, 0, 52)
DurContainer.LayoutOrder      = NextOrder()
DurContainer.BackgroundColor3 = Color3.fromRGB(16, 16, 22)
DurContainer.BorderSizePixel  = 0
Instance.new("UICorner", DurContainer)

local DurLabel = Instance.new("TextLabel", DurContainer)
DurLabel.Size                   = UDim2.new(1, -8, 0, 22)
DurLabel.Position               = UDim2.new(0, 4, 0, 2)
DurLabel.BackgroundTransparency = 1
DurLabel.TextColor3             = Color3.fromRGB(210, 210, 210)
DurLabel.Font                   = Enum.Font.GothamBold
DurLabel.TextSize               = 11
DurLabel.TextXAlignment         = Enum.TextXAlignment.Left
DurLabel.Text                   = "Duration: 2s"

local DurTrack = Instance.new("Frame", DurContainer)
DurTrack.Size             = UDim2.new(0.92, 0, 0, 7)
DurTrack.Position         = UDim2.new(0.04, 0, 0, 32)
DurTrack.BackgroundColor3 = Color3.fromRGB(38, 38, 50)
DurTrack.BorderSizePixel  = 0
Instance.new("UICorner", DurTrack)

local DurFill = Instance.new("Frame", DurTrack)
DurFill.Size             = UDim2.new(0.18, 0, 1, 0)
DurFill.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
DurFill.BorderSizePixel  = 0
Instance.new("UICorner", DurFill)
local dGrad = Instance.new("UIGradient", DurFill)
dGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(70, 70, 70)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 255, 255)),
})

local DurKnob = Instance.new("Frame", DurTrack)
local kSz = IsMobile and 18 or 13
DurKnob.Size             = UDim2.new(0, kSz, 0, kSz)
DurKnob.Position         = UDim2.new(0.18, -kSz/2, 0.5, -kSz/2)
DurKnob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
DurKnob.BorderSizePixel  = 0
Instance.new("UICorner", DurKnob)

local durLevels = {1, 1.5, 2, 2.5, 3, 4, 5, 7, 10}
local durIdx    = 3
Config.FlingDuration = durLevels[durIdx]

local sliderDrag = false
local function UpdateDurSlider(inp)
    local rel = math.clamp(
        (inp.Position.X - DurTrack.AbsolutePosition.X) / DurTrack.AbsoluteSize.X,
        0, 1
    )
    local idx = math.clamp(math.round(rel * (#durLevels - 1)) + 1, 1, #durLevels)
    durIdx               = idx
    Config.FlingDuration = durLevels[idx]
    local relPos         = (idx - 1) / (#durLevels - 1)
    DurFill.Size         = UDim2.new(relPos, 0, 1, 0)
    DurKnob.Position     = UDim2.new(relPos, -kSz/2, 0.5, -kSz/2)
    DurLabel.Text        = "Duration: " .. durLevels[idx] .. "s"
end

DurTrack.InputBegan:Connect(function(inp)
    if inp.UserInputType == Enum.UserInputType.MouseButton1
    or inp.UserInputType == Enum.UserInputType.Touch then
        sliderDrag = true
        UpdateDurSlider(inp)
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

-- WHITELIST
AddCat("WHITELIST")

local WLBtn    = MakeBtn("🛡  ADD CLOSEST TO WHITELIST")
local ClearBtn = MakeBtn("🗑  CLEAR WHITELIST", Color3.fromRGB(38, 18, 18))
ClearBtn.TextColor3 = Color3.fromRGB(220, 140, 140)

-- STATUS
AddCat("STATUS")

local StatusLbl = Instance.new("TextLabel", Scroll)
StatusLbl.Size                   = UDim2.new(0.97, 0, 0, 28)
StatusLbl.LayoutOrder            = NextOrder()
StatusLbl.BackgroundTransparency = 1
StatusLbl.Text                   = "● Idle"
StatusLbl.TextColor3             = Color3.fromRGB(110, 110, 120)
StatusLbl.Font                   = Enum.Font.GothamBold
StatusLbl.TextSize               = 13

local KeysLbl = Instance.new("TextLabel", Scroll)
KeysLbl.Size                   = UDim2.new(0.97, 0, 0, IsMobile and 36 or 22)
KeysLbl.LayoutOrder            = NextOrder()
KeysLbl.BackgroundTransparency = 1
KeysLbl.TextWrapped            = true
KeysLbl.Text                   = IsMobile
    and "F = відкрити меню  |  K = увімк/вимк"
    or  "J = UI  |  K = Toggle Fling"
KeysLbl.TextColor3             = Color3.fromRGB(60, 60, 75)
KeysLbl.Font                   = Enum.Font.Gotham
KeysLbl.TextSize               = IsMobile and 12 or 11

-- ============================================================
-- [[ ЛОГІКА КНОПОК ]]
-- ============================================================
local function UpdateStatus()
    if State.FlingLoop then
        StatusLbl.Text       = "● ACTIVE — флінгує"
        StatusLbl.TextColor3 = Color3.fromRGB(80, 255, 120)
        StartBtn.Text        = "■  STOP FLING"
        StartBtn.BackgroundColor3 = Color3.fromRGB(150, 28, 28)
    else
        StatusLbl.Text       = "● Idle"
        StatusLbl.TextColor3 = Color3.fromRGB(110, 110, 120)
        StartBtn.Text        = "▶  START FLING"
        StartBtn.BackgroundColor3 = Color3.fromRGB(22, 130, 40)
    end
end

StartBtn.MouseButton1Click:Connect(function()
    State.FlingLoop = not State.FlingLoop
    UpdateStatus()
    Notify("OMNI-FLING",
        State.FlingLoop and "Fling увімкнено ✓" or "Fling вимкнено ✗", 2)
end)

local styles     = {"auto", "up", "side", "down"}
local styleIdx2  = 1
local styleNames = {
    auto = "🌀  STYLE: AUTO (найсильніший)",
    up   = "⬆️  STYLE: UP",
    side = "↔️  STYLE: SIDE",
    down = "⬇️  STYLE: DOWN",
}

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

LP.CharacterAdded:Connect(function()
    State.FlingLoop  = false
    State.Flinging   = false
    workspace.FallenPartsDestroyHeight = getgenv().FPDH
    UpdateStatus()
end)

-- ============================================================
-- [[ DRAGGABLE ]]
-- ============================================================
do
    local drag, dStart, dPos = false, nil, nil
    TitleBar.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            drag   = true
            dStart = inp.Position
            dPos   = Main.Position
        end
    end)
    UIS.InputChanged:Connect(function(inp)
        if not drag then return end
        if inp.UserInputType == Enum.UserInputType.MouseMovement
        or inp.UserInputType == Enum.UserInputType.Touch then
            local d = inp.Position - dStart
            Main.Position = UDim2.new(
                dPos.X.Scale, dPos.X.Offset + d.X,
                dPos.Y.Scale, dPos.Y.Offset + d.Y
            )
        end
    end)
    UIS.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            drag = false
        end
    end)
end

-- ============================================================
-- [[ F КНОПКА ]]
-- ============================================================
local MBtnSz = IsMobile and 56 or 44

local MBtn = Instance.new("TextButton", Screen)
MBtn.Size             = UDim2.new(0, MBtnSz, 0, MBtnSz)
MBtn.Position         = UDim2.new(0, 10, 0.38, 0)
MBtn.Text             = "F"
MBtn.Font             = Enum.Font.GothamBlack
MBtn.TextSize         = IsMobile and 26 or 20
MBtn.BackgroundColor3 = Color3.fromRGB(10, 10, 15)
MBtn.TextColor3       = Color3.fromRGB(255, 255, 255)
MBtn.BorderSizePixel  = 0
MBtn.AutoButtonColor  = false
MBtn.ZIndex           = 100
Instance.new("UICorner", MBtn)
local mSt = Instance.new("UIStroke", MBtn)
mSt.Color = Color3.fromRGB(255, 255, 255); mSt.Thickness = 2

-- K кнопка (мобіль)
if IsMobile then
    local KBtn = Instance.new("TextButton", Screen)
    KBtn.Size             = UDim2.new(0, MBtnSz, 0, MBtnSz)
    KBtn.Position         = UDim2.new(0, 10, 0.50, 0)
    KBtn.Text             = "K"
    KBtn.Font             = Enum.Font.GothamBlack
    KBtn.TextSize         = 26
    KBtn.BackgroundColor3 = Color3.fromRGB(22, 130, 40)
    KBtn.TextColor3       = Color3.fromRGB(255, 255, 255)
    KBtn.BorderSizePixel  = 0
    KBtn.AutoButtonColor  = false
    KBtn.ZIndex           = 100
    Instance.new("UICorner", KBtn)
    local kSt2 = Instance.new("UIStroke", KBtn)
    kSt2.Color = Color3.fromRGB(255, 255, 255); kSt2.Thickness = 2

    KBtn.MouseButton1Click:Connect(function()
        State.FlingLoop = not State.FlingLoop
        UpdateStatus()
        KBtn.BackgroundColor3 = State.FlingLoop
            and Color3.fromRGB(150, 28, 28)
            or  Color3.fromRGB(22, 130, 40)
        Notify("OMNI-FLING",
            State.FlingLoop and "Fling ON ✓" or "Fling OFF ✗", 1.5)
    end)

    -- Drag K кнопки
    do
        local d2, s2, p2, m2, t2 = false, nil, nil, false, 0
        KBtn.InputBegan:Connect(function(inp)
            if inp.UserInputType == Enum.UserInputType.Touch then
                d2 = true; s2 = inp.Position; p2 = KBtn.Position
                t2 = tick(); m2 = false
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
    end
end

-- Drag F кнопки
do
    local mDrag, mStart, mPos = false, nil, nil
    local mTick, mMoved = 0, false
    MBtn.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            mDrag = true; mStart = inp.Position
            mPos  = MBtn.Position; mTick = tick(); mMoved = false
        end
    end)
    UIS.InputChanged:Connect(function(inp)
        if not mDrag then return end
        if inp.UserInputType == Enum.UserInputType.MouseMovement
        or inp.UserInputType == Enum.UserInputType.Touch then
            local d = inp.Position - mStart
            if d.Magnitude > 6 then mMoved = true end
            MBtn.Position = UDim2.new(
                mPos.X.Scale, mPos.X.Offset + d.X,
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
-- [[ B&W АНІМАЦІЯ ]]
-- ============================================================
task.spawn(function()
    local t = 0
    while true do
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
            ColorSequenceKeypoint.new(0,   Color3.fromRGB(vA, vA, vA)),
            ColorSequenceKeypoint.new(0.5, Color3.fromRGB(vB, vB, vB)),
            ColorSequenceKeypoint.new(1,   Color3.fromRGB(vA, vA, vA)),
        })
        TitleGrad.Rotation  = (t * 30) % 360
        local ts            = math.floor(180 + v * 75)
        TitleLbl.TextColor3 = Color3.fromRGB(ts, ts, ts)
    end
end)

-- ============================================================
-- [[ КЛАВІШІ ]]
-- ============================================================
UIS.InputBegan:Connect(function(inp, gpe)
    if gpe then return end
    if inp.KeyCode == Enum.KeyCode.J then
        Main.Visible = not Main.Visible
    end
    if inp.KeyCode == Enum.KeyCode.K then
        State.FlingLoop = not State.FlingLoop
        UpdateStatus()
        Notify("OMNI-FLING",
            State.FlingLoop and "Fling ON ✓" or "Fling OFF ✗", 1.5)
    end
end)

-- ============================================================
Notify("OMNI-FLING V260.51",
    IsMobile and "F=меню | K=toggle | SkidFling ✓"
             or  "J=меню | K=toggle | SkidFling ✓", 5)
