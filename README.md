-- Observer And run modifier
-- Efeito visual local para Roblox

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local SoundService = game:GetService("SoundService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- IDs dos assets
local OBSERVER_IMAGE = "rbxassetid://134660978043984"
local BLACK_IMAGE = "rbxassetid://131457731811137"
local MOVEMENT_SOUND = "rbxassetid://138798991855843"
local SCREAM_SOUND = "rbxassetid://138587239935890"

-- Configurações
local IMAGE_WIDTH = 340
local IMAGE_HEIGHT = 220
local INITIAL_SPEED = 0.05
local MAX_SPEED = 0.20
local SPEED_RAMP_TIME = 30
local RETURN_TIME = 0.22

-- Limpar execução anterior
local oldGui = playerGui:FindFirstChild("ObserverModifierGui")
if oldGui then
    oldGui:Destroy()
end

local oldSounds = SoundService:FindFirstChild(
    "ObserverModifierSounds"
)
if oldSounds then
    oldSounds:Destroy()
end

-- Interface
local gui = Instance.new("ScreenGui")
gui.Name = "ObserverModifierGui"
gui.IgnoreGuiInset = true
gui.ResetOnSpawn = false
gui.DisplayOrder = 999999
gui.Parent = playerGui

-- Título (desaparece após 3 segundos)
local title = Instance.new("TextLabel")
title.Name = "Title"
title.Size = UDim2.new(1, 0, 0, 45)
title.Position = UDim2.new(0, 0, 0, 10)
title.BackgroundTransparency = 1
title.Text = "Observer And run modifier"
title.TextColor3 = Color3.new(1, 1, 1)
title.TextStrokeTransparency = 0
title.TextScaled = true
title.Font = Enum.Font.GothamBold
title.Parent = gui

task.delay(3, function()
    if title.Parent then
        title:Destroy()
    end
end)

-- Pasta de sons
local soundFolder = Instance.new("Folder")
soundFolder.Name = "ObserverModifierSounds"
soundFolder.Parent = SoundService

-- Som de quando o jogador fica parado
local movementSound = Instance.new("Sound")
movementSound.Name = "ObserverLoop"
movementSound.SoundId = MOVEMENT_SOUND
movementSound.Looped = true
movementSound.Volume = 1
movementSound.Parent = soundFolder

-- Som do jumpscare
local scream = Instance.new("Sound")
scream.Name = "ObserverScream"
scream.SoundId = SCREAM_SOUND
scream.Looped = true
scream.Volume = 1
scream.Parent = soundFolder

-- Observer
local observer = Instance.new("ImageLabel")
observer.Name = "Observer"
observer.BackgroundTransparency = 1
observer.Image = OBSERVER_IMAGE
observer.AnchorPoint = Vector2.new(0.5, 0.5)
observer.Size = UDim2.fromOffset(IMAGE_WIDTH, IMAGE_HEIGHT)
observer.Position = UDim2.new(
    0.5, 0, 0, -IMAGE_HEIGHT / 2
)
observer.Rotation = 180
observer.ScaleType = Enum.ScaleType.Fit
observer.Parent = gui

-- Tela preta com imagem do jumpscare
local blackScreen = Instance.new("ImageLabel")
blackScreen.Name = "Jumpscare"
blackScreen.BackgroundColor3 = Color3.new(0, 0, 0)
blackScreen.BackgroundTransparency = 0
blackScreen.Image = BLACK_IMAGE
blackScreen.ImageTransparency = 0
blackScreen.Size = UDim2.fromScale(1, 1)
blackScreen.Position = UDim2.fromScale(0, 0)
blackScreen.ScaleType = Enum.ScaleType.Stretch
blackScreen.Visible = false
blackScreen.ZIndex = 100
blackScreen.Parent = gui

-- Estado
local character
local humanoid
local root
local lastPosition
local stillTime = 0
local progress = 0
local triggered = false
local dead = false
local returning = false
local returnTween

local function cleanup()
    if dead then
        return
    end

    dead = true

    if returnTween then
        returnTween:Cancel()
    end

    movementSound:Stop()
    scream:Stop()

    if gui.Parent then
        gui:Destroy()
    end

    if soundFolder.Parent then
        soundFolder:Destroy()
    end
end

local function setupCharacter(char)
    character = char
    humanoid = char:WaitForChild("Humanoid")
    root = char:WaitForChild("HumanoidRootPart")
    lastPosition = root.Position

    stillTime = 0
    progress = 0
    triggered = false
    returning = false

    movementSound:Stop()
    scream:Stop()

    observer.Visible = true
    blackScreen.Visible = false
    observer.Position = UDim2.new(
        0.5, 0, 0, -IMAGE_HEIGHT / 2
    )

    humanoid.Died:Connect(cleanup)
end

if player.Character then
    setupCharacter(player.Character)
end

player.CharacterAdded:Connect(function(char)
    if not dead and gui.Parent then
        setupCharacter(char)
    end
end)

-- Atualização do Observer
local connection
connection = RunService.RenderStepped:Connect(function(dt)
    if dead or not gui.Parent then
        connection:Disconnect()
        return
    end

    if triggered then
        return
    end

    if not character or not humanoid or not root
        or humanoid.Health <= 0 then
        return
    end

    local currentPosition = root.Position
    local moved = (
        currentPosition - lastPosition
    ).Magnitude > 0.05

    lastPosition = currentPosition

    if moved then
        -- Ao andar, o Observer volta rapidamente ao topo
        stillTime = 0
        progress = 0
        movementSound:Stop()

        if not returning then
            returning = true

            if returnTween then
                returnTween:Cancel()
            end

            returnTween = TweenService:Create(
                observer,
                TweenInfo.new(
                    RETURN_TIME,
                    Enum.EasingStyle.Quad,
                    Enum.EasingDirection.Out
                ),
                {
                    Position = UDim2.new(
                        0.5, 0, 0, -IMAGE_HEIGHT / 2
                    )
                }
            )

            returnTween:Play()

            task.spawn(function()
                returnTween.Completed:Wait()
                returning = false
            end)
        end
    else
        if returning then
            return
        end

        stillTime += dt

        if not movementSound.IsPlaying then
            movementSound:Play()
        end

        -- A velocidade aumenta de 5% até 20%
        local ramp = math.clamp(
            stillTime / SPEED_RAMP_TIME, 0, 1
        )

        local speed = INITIAL_SPEED
            + (MAX_SPEED - INITIAL_SPEED) * ramp

        progress = math.clamp(
            progress + speed * dt, 0, 1
        )

        -- O Observer para perto do topo, não no fundo
        local camera = workspace.CurrentCamera
        local screenHeight = camera
            and camera.ViewportSize.Y or 768

        local endY = math.min(
            IMAGE_HEIGHT / 2,
            screenHeight * 0.13
        )

        local startY = -IMAGE_HEIGHT / 2
        local currentY = startY
            + (endY - startY) * progress

        observer.Position = UDim2.new(
            0.5, 0, 0, currentY
        )

        -- Jumpscare
        if progress >= 1 then
            triggered = true
            movementSound:Stop()
            observer.Visible = false
            blackScreen.Visible = true
            scream:Play()
        end
    end
end)
