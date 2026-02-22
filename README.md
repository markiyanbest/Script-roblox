-- [[ V260.49: OMNI-FLING - PERFECT EDITION ]]
-- [[ FIXES: ANTI-KICK | DRAGGING | CLEANUP | STABILITY ]]

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS        = game:GetService("UserInputService")
local Workspace  = game:GetService("Workspace")
local StarterGui = game:GetService("StarterGui")

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

-- Зберігаємо оригінальне значення
local OrigFPDH = workspace.FallenPartsDestroyHeight

local Whitelist = { LP.UserId }

local State = {
    FlingLoop   = false,
    FlingStyle  = "infinite", -- "up" | "side" | "infinite"
    Flinging    = false,      -- захист від подвійного виклику
}

local Config = {
    CurrentTarget = "",       -- порожньо = найближчий
}

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

-- Перевірка чи гравець у whitelist
local function IsWhitelisted(player)
    for _, id in pairs(Whitelist) do
        if player.UserId == id then return true end
    end
    return false
end

-- Перевірка чи персонаж живий
local function IsAlive(player)
    if not player or not player.Character then return false end
    local hum = player.Character:FindFirstChildOfClass("Humanoid")
    return hum and hum.Health > 0
end

-- Найближчий гравець по дистанції
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
            if d < bestDist then
                bestDist = d
                best     = v
            end
        end
    end
    return best
end

-- Пошук гравця по імені/команді
local function GetTarget(input)
    input = (input or ""):lower():match("^%s*(.-)%s*$") -- trim

    -- Порожньо або пробіл = найближчий
    if input == "" or input == " " then
        return GetClosestPlayer()
    end

    if input == "all" then return "all" end

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

    -- Пошук по імені/displayname
    for _, v in pairs(Players:GetPlayers()) do
        if v == LP then continue end
        if v.Name:lower():find(input, 1, true)
        or v.DisplayName:lower():find(input, 1, true) then
            return v
        end
    end

    return nil
end

-- ============================================================
-- [[ 3. FLING ЛОГІКА ]]
-- ============================================================

local function GetFlingOffset()
    if State.FlingStyle == "up" then
        return CFrame.new(0, 2, 0)
    elseif State.FlingStyle == "side" then
        return CFrame.new(3, 0, 0)
    else -- infinite
        return CFrame.new(
            math.random(-15, 15) / 10,
            math.random(-15, 15) / 10,
            math.random(-15, 15) / 10
        )
    end
end

local function FlingPlayer(target)
    if not target or not State.FlingLoop then return end
    if IsWhitelisted(target) then return end
    if not IsAlive(target) then return end
    if State.Flinging then return end -- захист від подвійного виклику

    local myChar = LP.Character
    local myHum  = myChar and myChar:FindFirstChildOfClass("Humanoid")
    local myHRP  = myChar and myChar:FindFirstChild("HumanoidRootPart")

    local tChar = target.Character
    local tHum  = tChar and tChar:FindFirstChildOfClass("Humanoid")
    local tHRP  = tChar and tChar:FindFirstChild("HumanoidRootPart")

    if not myHRP or not tHRP or not myHum or not tHum then return end

    State.Flinging = true

    -- Зберігаємо позицію для відновлення
    local savedCFrame = myHRP.CFrame
    local savedCamSubject = Camera.CameraSubject

    -- Розблоковуємо висоту падіння
    workspace.FallenPartsDestroyHeight = -math.huge

    -- Форс-встаємо якщо сидимо
    myHum:SetStateEnabled(Enum.HumanoidStateType.Seated, false)
    if myHum.Sit then myHum.Sit = false end

    -- Камера на ціль
    Camera.CameraSubject = tHRP

    -- BodyVelocity для нашого персонажа
    local bv = Instance.new("BodyVelocity")
    bv.Velocity  = Vector3.new(9e8, 9e8, 9e8)
    bv.MaxForce  = Vector3.new(math.huge, math.huge, math.huge)
    bv.Parent    = myHRP

    -- BodyAngularVelocity для обертання
    local bav = Instance.new("BodyAngularVelocity")
    bav.AngularVelocity = Vector3.new(9e8, 9e8, 9e8)
    bav.MaxTorque       = Vector3.new(math.huge, math.huge, math.huge)
    bav.Parent          = myHRP

    local startTime = tick()
    local duration  = 1.2 -- секунди флінгу

    -- Основний цикл флінгу
    while State.FlingLoop
    and IsAlive(target)
    and IsAlive(LP)
    and tick() - startTime < duration do

        -- Телепортуємось до цілі з офсетом
        local offset = GetFlingOffset()
        local angles = CFrame.Angles(
            math.rad(math.random(0, 360)),
            math.rad(math.random(0, 360)),
            math.rad(math.random(0, 360))
        )
        myHRP.CFrame = tHRP.CFrame * offset * angles

        -- Швидкість (безпечний рівень щоб не кікало)
        local safeVel = Vector3.new(
            math.random(800, 1200),
            math.random(800, 1200),
            math.random(800, 1200)
        )
        myHRP.AssemblyLinearVelocity  = safeVel
        myHRP.AssemblyAngularVelocity = Vector3.new(
            math.random(100, 300),
            math.random(100, 300),
            math.random(100, 300)
        )

        -- Рандомна затримка між тіками (анти-детект)
        task.wait(math.random(6, 14) / 1000)
    end

    -- Cleanup
    pcall(function() bv:Destroy()  end)
    pcall(function() bav:Destroy() end)

    -- Відновлення стану
    myHUM = myChar:FindFirstChildOfClass("Humanoid")
    if myHUM then
        myHUM:SetStateEnabled(Enum.HumanoidStateType.Seated, true)
    end

    -- Відновлення позиції
    pcall(function()
        myHRP.AssemblyLinearVelocity  = Vector3.zero
        myHRP.AssemblyAngularVelocity = Vector3.zero
        myHRP.CFrame = savedCFrame
    end)

    -- Відновлення камери
    pcall(function()
        Camera.CameraSubject = savedCamSubject
            or (LP.Character and LP.Character:FindFirstChildOfClass("Humanoid"))
    end)

    -- Відновлення висоти падіння
    workspace.FallenPartsDestroyHeight = OrigFPDH

    State.Flinging = false
end

-- ============================================================
-- [[ 4. ГОЛОВНИЙ ЦИКЛ ]]
-- ============================================================
task.spawn(function()
    while true do
        -- Затримка між циклами (анти-кік + анти-детект)
        task.wait(math.random(8, 18) / 100)

        if not State.FlingLoop then continue end
        if not IsAlive(LP) then continue end
        if State.Flinging then continue end

        local target = GetTarget(Config.CurrentTarget)

        if target == "all" then
            -- Флінг всіх по черзі
            for _, v in pairs(Players:GetPlayers()) do
                if not State.FlingLoop then break end
                if v == LP or IsWhitelisted(v) then continue end
                if not IsAlive(v) then continue end
                FlingPlayer(v)
                task.wait(math.random(5, 12) / 100)
            end
        elseif target then
            FlingPlayer(target)
        else
            -- Якщо ціль не знайдена — чекаємо
            task.wait(0.5)
        end
    end
end)

-- Cleanup при смерті/респавні
LP.CharacterAdded:Connect(function()
    State.FlingLoop  = false
    State.Flinging   = false
    workspace.FallenPartsDestroyHeight = OrigFPDH
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
Screen.Name         = "OmniFling"
Screen.ResetOnSpawn = false

local IsMobile = UIS.TouchEnabled
local W, H     = 230, 330

-- Головний фрейм
local Main = Instance.new("Frame", Screen)
Main.Size             = UDim2.new(0, W, 0, H)
Main.Position         = UDim2.new(0.5, -W/2, 0.5, -H/2)
Main.BackgroundColor3 = Color3.fromRGB(12, 12, 16)
Main.BorderSizePixel  = 0
Main.Active           = false -- НЕ використовуємо Draggable (Shift Lock баг)
Instance.new("UICorner", Main)
local ms = Instance.new("UIStroke", Main)
ms.Color = Color3.fromRGB(255, 255, 255); ms.Thickness = 1.5

-- Заголовок (використовується для перетягування)
local TitleBar = Instance.new("Frame", Main)
TitleBar.Size             = UDim2.new(1, 0, 0, 38)
TitleBar.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
TitleBar.BorderSizePixel  = 0
Instance.new("UICorner", TitleBar)

local TitleLbl = Instance.new("TextLabel", TitleBar)
TitleLbl.Size               = UDim2.new(1, 0, 1, 0)
TitleLbl.BackgroundTransparency = 1
TitleLbl.TextColor3         = Color3.fromRGB(255, 255, 255)
TitleLbl.Font               = Enum.Font.GothamBlack
TitleLbl.TextSize            = 15
TitleLbl.Text               = "⚡ OMNI-FLING V260.49"

-- Layout
local Layout = Instance.new("UIListLayout", Main)
Layout.Padding             = UDim.new(0, 6)
Layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
Layout.SortOrder           = Enum.SortOrder.LayoutOrder

local Pad = Instance.new("UIPadding", Main)
Pad.PaddingTop    = UDim.new(0, 44)
Pad.PaddingLeft   = UDim.new(0, 8)
Pad.PaddingRight  = UDim.new(0, 8)
Pad.PaddingBottom = UDim.new(0, 8)

-- ============================================================
-- [[ ПЕРЕТЯГУВАННЯ (ЗАХИСТ ВІД SHIFT LOCK) ]]
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
-- [[ UI КОМПОНЕНТИ ]]
-- ============================================================

-- TextBox для цілі
local TargetBox = Instance.new("TextBox", Main)
TargetBox.Size               = UDim2.new(1, 0, 0, 36)
TargetBox.LayoutOrder        = 1
TargetBox.PlaceholderText    = "Target / all / empty=closest"
TargetBox.Text               = ""
TargetBox.BackgroundColor3   = Color3.fromRGB(28, 28, 36)
TargetBox.TextColor3         = Color3.fromRGB(255, 255, 255)
TargetBox.PlaceholderColor3  = Color3.fromRGB(120, 120, 130)
TargetBox.Font               = Enum.Font.GothamBold
TargetBox.TextSize            = 13
TargetBox.ClearTextOnFocus   = false
TargetBox.BorderSizePixel    = 0
Instance.new("UICorner", TargetBox)

TargetBox.FocusLost:Connect(function()
    Config.CurrentTarget = TargetBox.Text
end)

-- Кнопка START/STOP
local StartBtn = Instance.new("TextButton", Main)
StartBtn.Size             = UDim2.new(1, 0, 0, 48)
StartBtn.LayoutOrder      = 2
StartBtn.Text             = "▶ START FLING"
StartBtn.BackgroundColor3 = Color3.fromRGB(30, 160, 50)
StartBtn.TextColor3       = Color3.fromRGB(255, 255, 255)
StartBtn.Font             = Enum.Font.GothamBlack
StartBtn.TextSize          = 15
StartBtn.BorderSizePixel  = 0
StartBtn.AutoButtonColor  = false
Instance.new("UICorner", StartBtn)

-- Кнопка STYLE
local StyleBtn = Instance.new("TextButton", Main)
StyleBtn.Size             = UDim2.new(1, 0, 0, 38)
StyleBtn.LayoutOrder      = 3
StyleBtn.Text             = "🔄 STYLE: INFINITE"
StyleBtn.BackgroundColor3 = Color3.fromRGB(45, 45, 58)
StyleBtn.TextColor3       = Color3.fromRGB(255, 255, 255)
StyleBtn.Font             = Enum.Font.GothamBold
StyleBtn.TextSize          = 13
StyleBtn.BorderSizePixel  = 0
StyleBtn.AutoButtonColor  = false
Instance.new("UICorner", StyleBtn)

-- Whitelist кнопка
local WLBtn = Instance.new("TextButton", Main)
WLBtn.Size             = UDim2.new(1, 0, 0, 36)
WLBtn.LayoutOrder      = 4
WLBtn.Text             = "🛡 ADD CLOSEST TO WHITELIST"
WLBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
WLBtn.TextColor3       = Color3.fromRGB(200, 200, 255)
WLBtn.Font             = Enum.Font.GothamBold
WLBtn.TextSize          = 11
WLBtn.BorderSizePixel  = 0
WLBtn.AutoButtonColor  = false
Instance.new("UICorner", WLBtn)

-- Статус
local StatusLbl = Instance.new("TextLabel", Main)
StatusLbl.Size               = UDim2.new(1, 0, 0, 28)
StatusLbl.LayoutOrder        = 5
StatusLbl.Text               = "● Status: Idle"
StatusLbl.TextColor3         = Color3.fromRGB(130, 130, 140)
StatusLbl.BackgroundTransparency = 1
StatusLbl.Font               = Enum.Font.GothamBold
StatusLbl.TextSize            = 13
StatusLbl.BorderSizePixel    = 0

-- Інфо про біндлавіші
local InfoLbl = Instance.new("TextLabel", Main)
InfoLbl.Size               = UDim2.new(1, 0, 0, 22)
InfoLbl.LayoutOrder        = 6
InfoLbl.Text               = "J = Toggle UI  |  K = Toggle Fling"
InfoLbl.TextColor3         = Color3.fromRGB(80, 80, 90)
InfoLbl.BackgroundTransparency = 1
InfoLbl.Font               = Enum.Font.Gotham
InfoLbl.TextSize            = 11
InfoLbl.BorderSizePixel    = 0

-- ============================================================
-- [[ ЛОГІКА КНОПОК ]]
-- ============================================================

-- Оновлення статусу
local function UpdateStatus()
    if State.FlingLoop then
        StatusLbl.Text      = "● Status: ACTIVE"
        StatusLbl.TextColor3 = Color3.fromRGB(80, 255, 120)
        StartBtn.Text       = "■ STOP FLING"
        StartBtn.BackgroundColor3 = Color3.fromRGB(180, 35, 35)
    else
        StatusLbl.Text      = "● Status: Idle"
        StatusLbl.TextColor3 = Color3.fromRGB(130, 130, 140)
        StartBtn.Text       = "▶ START FLING"
        StartBtn.BackgroundColor3 = Color3.fromRGB(30, 160, 50)
    end
end

-- Start/Stop
StartBtn.MouseButton1Click:Connect(function()
    State.FlingLoop = not State.FlingLoop
    UpdateStatus()
    Notify(
        "OMNI-FLING",
        State.FlingLoop and "Fling увімкнено ✓" or "Fling вимкнено ✗",
        2
    )
end)

-- Style selector
local styles = {"infinite", "up", "side"}
local styleIdx = 1
local styleNames = {
    infinite = "🌀 STYLE: INFINITE",
    up       = "⬆️ STYLE: UP",
    side     = "↔️ STYLE: SIDE",
}

StyleBtn.MouseButton1Click:Connect(function()
    styleIdx = (styleIdx % #styles) + 1
    State.FlingStyle = styles[styleIdx]
    StyleBtn.Text = styleNames[State.FlingStyle]
end)

-- Whitelist
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

-- Respawn cleanup
LP.CharacterAdded:Connect(function()
    State.FlingLoop  = false
    State.Flinging   = false
    workspace.FallenPartsDestroyHeight = OrigFPDH
    UpdateStatus()
end)

-- ============================================================
-- [[ M/J КНОПКА (ПЕРЕТЯГУВАНА) ]]
-- ============================================================
local MBtnSize = IsMobile and 52 or 44

local MBtn = Instance.new("TextButton", Screen)
MBtn.Size             = UDim2.new(0, MBtnSize, 0, MBtnSize)
MBtn.Position         = UDim2.new(0, 10, 0.45, 0)
MBtn.Text             = "F"
MBtn.Font             = Enum.Font.GothamBlack
MBtn.TextSize          = IsMobile and 24 or 20
MBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
MBtn.TextColor3       = Color3.fromRGB(255, 255, 255)
MBtn.BorderSizePixel  = 0
MBtn.AutoButtonColor  = false
MBtn.ZIndex           = 100
Instance.new("UICorner", MBtn)
local mStroke = Instance.new("UIStroke", MBtn)
mStroke.Color = Color3.fromRGB(255, 255, 255); mStroke.Thickness = 1.5

-- Перетягування M кнопки з захистом від випадкового кліку
do
    local mDrag, mStart, mPos = false, nil, nil
    local mTick, mMoved = 0, false

    MBtn.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            mDrag  = true
            mStart = inp.Position
            mPos   = MBtn.Position
            mTick  = tick()
            mMoved = false
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
                mPos.Y.Scale, mPos.Y.Offset + d.Y
            )
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
-- [[ КЛАВІШІ ]]
-- ============================================================
UIS.InputBegan:Connect(function(inp, gpe)
    if gpe then return end

    -- J = Toggle UI
    if inp.KeyCode == Enum.KeyCode.J then
        Main.Visible = not Main.Visible
    end

    -- K = Toggle Fling (швидкий бінд без відкриття меню)
    if inp.KeyCode == Enum.KeyCode.K then
        State.FlingLoop = not State.FlingLoop
        UpdateStatus()
        Notify(
            "OMNI-FLING",
            State.FlingLoop and "Fling ON ✓" or "Fling OFF ✗",
            1.5
        )
    end
end)

-- ============================================================
Notify("OMNI-FLING V260.49", "J=меню | K=toggle | F=кнопка меню", 5)
