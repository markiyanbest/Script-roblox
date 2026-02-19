-- [[ V260.48: OMNI-FLING MASTER - THE FINAL 10/10 VERSION ]]
-- [[ NO COMPRESSION | ANTI-KICK | STYLE SELECTOR | FULL BYPASS ]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local StarterGui = game:GetService("StarterGui")

local LP = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- [[ ГЛОБАЛЬНІ НАЛАШТУВАННЯ ]]
getgenv().FPDH = workspace.FallenPartsDestroyHeight
local Whitelist = {1414978355, LP.UserId}

local State = {
    FlingLoop = false,
    MenuVisible = true,
    FlingStyle = "infinite" -- "up", "side", "infinite"
}

local Config = {
    CurrentTarget = "All"
}

-- [[ ПОВІДОМЛЕННЯ ]]
local function Message(_Title, _Text, Time)
    StarterGui:SetCore("SendNotification", {
        Title = _Title, 
        Text = _Text, 
        Duration = Time or 5
    })
end

-- [[ ПОШУК НАЙБЛИЖЧОГО ]]
local function GetClosestPlayer()
    local closest = nil
    local dist = math.huge
    for _, v in pairs(Players:GetPlayers()) do
        if v ~= LP and v.Character and v.Character:FindFirstChild("HumanoidRootPart") then
            local magnitude = (v.Character.HumanoidRootPart.Position - LP.Character.HumanoidRootPart.Position).Magnitude
            if magnitude < dist then
                dist = magnitude
                closest = v
            end
        end
    end
    return closest
end

-- [[ ПОШУК ЦІЛІ ]]
local function GetPlayer(Name)
    if Name == "" or Name == " " then return GetClosestPlayer() end
    Name = Name:lower()
    if Name == "all" then return "all" end
    if Name == "random" then
        local plrs = Players:GetPlayers()
        table.remove(plrs, table.find(plrs, LP))
        return #plrs > 0 and plrs[math.random(#plrs)] or nil
    end
    for _, x in next, Players:GetPlayers() do
        if x ~= LP and (x.Name:lower():match("^"..Name) or x.DisplayName:lower():match("^"..Name)) then
            return x
        end
    end
    return nil
end

-- [[ ОСНОВНИЙ ФЛІНГ ]]
local function SkidFling(TargetPlayer)
    if not TargetPlayer or not State.FlingLoop then return end
    if table.find(Whitelist, TargetPlayer.UserId) then return end

    local Char = LP.Character
    local Hum = Char and Char:FindFirstChildOfClass("Humanoid")
    local Root = Hum and Hum.RootPart

    local TChar = TargetPlayer.Character
    local THum = TChar and TChar:FindFirstChildOfClass("Humanoid")
    local TRoot = THum and THum.RootPart

    if Root and TRoot then
        -- Anti-sit bypass (Примусово змушуємо ціль встати)
        if THum and THum.Sit then
            THum.Sit = false
            task.wait(0.1)
        end

        local OldCFrame = Root.CFrame
        workspace.FallenPartsDestroyHeight = 0/0
        
        Message("Fling", "Targeting: " .. TargetPlayer.Name, 2)

        local BV = Instance.new("BodyVelocity", Root)
        BV.Velocity = Vector3.new(9e8, 9e8, 9e8)
        BV.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        
        Hum:SetStateEnabled(Enum.HumanoidStateType.Seated, false)
        Camera.CameraSubject = TRoot

        local Time = tick()
        repeat
            if not State.FlingLoop or not TRoot then break end
            
            local Offset = CFrame.new(0, 0, 0)
            if State.FlingStyle == "up" then
                Offset = CFrame.new(0, 1.5, 0)
            elseif State.FlingStyle == "side" then
                Offset = CFrame.new(2.5, 0, 0)
            elseif State.FlingStyle == "infinite" then
                Offset = CFrame.new(math.random(-10,10)/10, math.random(-10,10)/10, math.random(-10,10)/10)
            end

            Root.CFrame = TRoot.CFrame * Offset * CFrame.Angles(math.rad(math.random(0,360)), math.rad(math.random(0,360)), math.rad(math.random(0,360)))
            
            -- Velocity Cap (Обмеження швидкості, щоб не кікало)
            local targetVel = Vector3.new(9e7, 9e7, 9e7)
            if targetVel.Magnitude > 25 then
                Root.Velocity = targetVel.Unit * 24 -- Тримаємо швидкість у безпечних межах
            else
                Root.Velocity = targetVel
            end
            
            Root.RotVelocity = Vector3.new(9e8, 9e8, 9e8)
            
            -- Random delay між тіками (Анти-детект)
            task.wait(math.random(5, 15) / 1000) 
        until not TargetPlayer.Parent or Hum.Health <= 0 or tick() > Time + 1.3 or not State.FlingLoop

        BV:Destroy()
        Root.Velocity = Vector3.zero
        Root.RotVelocity = Vector3.zero
        Hum:SetStateEnabled(Enum.HumanoidStateType.Seated, true)
        Camera.CameraSubject = Hum
        Root.CFrame = OldCFrame
        workspace.FallenPartsDestroyHeight = getgenv().FPDH
    end
end

-- [[ UI СИСТЕМА ]]
local function CreateUI()
    pcall(function() game:GetService("CoreGui").Omni10X:Destroy() end)
    local Screen = Instance.new("ScreenGui", game:GetService("CoreGui"))
    Screen.Name = "Omni10X"
    
    local Main = Instance.new("Frame", Screen)
    Main.Size = UDim2.new(0, 220, 0, 310); Main.Position = UDim2.new(0.5, -110, 0.5, -155)
    Main.BackgroundColor3 = Color3.fromRGB(15, 15, 15); Main.Active = true; Main.Draggable = true
    Instance.new("UICorner", Main)
    Instance.new("UIStroke", Main).Color = Color3.new(1, 1, 1)

    local Title = Instance.new("TextLabel", Main)
    Title.Size = UDim2.new(1, 0, 0, 40); Title.Text = "OMNI-FLING 10/10"; Title.TextColor3 = Color3.new(1,1,1)
    Title.BackgroundTransparency = 1; Title.Font = Enum.Font.GothamBold

    local Txt = Instance.new("TextBox", Main)
    Txt.Size = UDim2.new(0.9, 0, 0, 40); Txt.Position = UDim2.new(0.05, 0, 0, 45)
    Txt.PlaceholderText = "Target / All / Empty=Closest"; Txt.Text = ""
    Txt.BackgroundColor3 = Color3.fromRGB(30, 30, 35); Txt.TextColor3 = Color3.new(1,1,1)
    Instance.new("UICorner", Txt)
    Txt.FocusLost:Connect(function() Config.CurrentTarget = Txt.Text end)

    local StartBtn = Instance.new("TextButton", Main)
    StartBtn.Size = UDim2.new(0.9, 0, 0, 50); StartBtn.Position = UDim2.new(0.05, 0, 0, 95)
    StartBtn.Text = "START MASTER FLING"; StartBtn.BackgroundColor3 = Color3.fromRGB(40, 150, 40)
    StartBtn.TextColor3 = Color3.new(1,1,1); Instance.new("UICorner", StartBtn); StartBtn.Font = Enum.Font.GothamBold

    local StyleBtn = Instance.new("TextButton", Main)
    StyleBtn.Size = UDim2.new(0.9, 0, 0, 45); StyleBtn.Position = UDim2.new(0.05, 0, 0, 155)
    StyleBtn.Text = "STYLE: INFINITE"; StyleBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    StyleBtn.TextColor3 = Color3.new(1,1,1); Instance.new("UICorner", StyleBtn); StyleBtn.Font = Enum.Font.GothamBold

    StyleBtn.MouseButton1Click:Connect(function()
        if State.FlingStyle == "infinite" then State.FlingStyle = "up"
        elseif State.FlingStyle == "up" then State.FlingStyle = "side"
        else State.FlingStyle = "infinite" end
        StyleBtn.Text = "STYLE: " .. State.FlingStyle:upper()
    end)

    local Status = Instance.new("TextLabel", Main)
    Status.Size = UDim2.new(1, 0, 0, 30); Status.Position = UDim2.new(0, 0, 0, 210)
    Status.Text = "Status: Idle"; Status.TextColor3 = Color3.new(0.6, 0.6, 0.6)
    Status.BackgroundTransparency = 1; Status.Font = Enum.Font.Gotham

    StartBtn.MouseButton1Click:Connect(function()
        State.FlingLoop = not State.FlingLoop
        StartBtn.Text = State.FlingLoop and "STOP FLING" or "START MASTER FLING"
        StartBtn.BackgroundColor3 = State.FlingLoop and Color3.fromRGB(150, 40, 40) or Color3.fromRGB(40, 150, 40)
        Status.Text = State.FlingLoop and "Status: ACTIVE" or "Status: Idle"
    end)

    -- Auto-disable on death/respawn з повідомленням
    LP.CharacterAdded:Connect(function()
        State.FlingLoop = false
        StartBtn.Text = "START MASTER FLING"
        StartBtn.BackgroundColor3 = Color3.fromRGB(40, 150, 40)
        Status.Text = "Status: Idle (Respawned)"
        Message("Fling", "Disabled on respawn", 3)
    end)

    -- [[ КНОПКА ДЛЯ ТЕЛЕФОНІВ ]]
    local IsMobile = UIS.TouchEnabled and not UIS.KeyboardEnabled -- Перевірка, чи це телефон/планшет
    
    local MobileBtn = Instance.new("TextButton", Screen)
    MobileBtn.Name = "MobileToggle"
    MobileBtn.Size = UDim2.new(0, 50, 0, 50)
    MobileBtn.Position = UDim2.new(0, 10, 0.5, -25) -- Зліва по центру екрана
    MobileBtn.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
    MobileBtn.TextColor3 = Color3.new(1, 1, 1)
    MobileBtn.Text = "UI"
    MobileBtn.Font = Enum.Font.GothamBold
    MobileBtn.Visible = IsMobile -- Буде видимою лише на мобільних пристроях
    Instance.new("UICorner", MobileBtn).CornerRadius = UDim.new(0, 10)
    Instance.new("UIStroke", MobileBtn).Color = Color3.new(1, 1, 1)

    MobileBtn.MouseButton1Click:Connect(function()
        Main.Visible = not Main.Visible
    end)

    -- J-Toggle (Для ПК)
    UIS.InputBegan:Connect(function(i, g)
        if not g and i.KeyCode == Enum.KeyCode.J then Main.Visible = not Main.Visible end
    end)
end

-- [[ ГОЛОВНИЙ ЦИКЛ ]]
task.spawn(function()
    while true do
        -- Random delay між циклами флінгу (Анти-кік)
        task.wait(math.random(5, 15) / 100) 
        
        if State.FlingLoop then
            local t = GetPlayer(Config.CurrentTarget)
            if t == "all" then
                for _, v in pairs(Players:GetPlayers()) do
                    if not State.FlingLoop then break end
                    if v ~= LP then SkidFling(v) end
                end
            elseif t then
                SkidFling(t)
            end
        end
    end
end)

CreateUI()
Message("Omni-Fling 10/10", "Ready! Press 'J' or use UI button to Hide Menu.", 5)
