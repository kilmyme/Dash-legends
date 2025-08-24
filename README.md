local player = game.Players.LocalPlayer
local rs = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")

local DashPower = 100
local cooldownTime = 1
local maxDashCount = 2
local dashCount = 0
local lastJumpTime = 0
local maxJumpInterval = 0.35

local Animations = {
	Back = "rbxassetid://16000109129",
	Left = "rbxassetid://15999993442",
	Right = "rbxassetid://16000014797",
}

-- controle do toggle
local dashEnabled = true  

-- cria botão
local playerGui = player:WaitForChild("PlayerGui")
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "DashToggleGui"
screenGui.Parent = playerGui

local button = Instance.new("TextButton")
button.Size = UDim2.new(0, 25, 0, 25) -- bem menor
button.Position = UDim2.new(0.05, 0, 0.85, 0)
button.Text = "⚡"
button.TextColor3 = Color3.new(1, 1, 1)
button.BackgroundColor3 = Color3.fromRGB(0, 200, 0) -- verde (ativado por padrão)
button.Parent = screenGui
button.Font = Enum.Font.SourceSansBold
button.TextSize = 14
button.AutoButtonColor = true
Instance.new("UICorner", button)

-- arrastar botão
local dragging = false
local dragInput, dragStart, startPos

local function update(input)
	local delta = input.Position - dragStart
	button.Position = UDim2.new(
		startPos.X.Scale,
		startPos.X.Offset + delta.X,
		startPos.Y.Scale,
		startPos.Y.Offset + delta.Y
	)
end

button.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = input.Position
		startPos = button.Position

		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				dragging = false
			end
		end)
	end
end)

button.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		dragInput = input
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if input == dragInput and dragging then
		update(input)
	end
end)

-- clique para ativar/desativar
button.MouseButton1Click:Connect(function()
	dashEnabled = not dashEnabled
	if dashEnabled then
		button.BackgroundColor3 = Color3.fromRGB(0, 200, 0) -- verde ativado
	else
		button.BackgroundColor3 = Color3.fromRGB(200, 50, 50) -- vermelho desativado
	end
end)

-- função principal do dash
local function setupCharacter(char)
	local humanoid = char:WaitForChild("Humanoid")
	local hrp = char:WaitForChild("HumanoidRootPart")

	local cooldown = false

	local function getMoveDirection()
		local dir = humanoid.MoveDirection
		if dir.Magnitude > 0 then
			return dir.Unit
		end
		return nil
	end

	UserInputService.JumpRequest:Connect(function()
		if not dashEnabled then return end -- botão desativado
		if cooldown then return end

		local moveDir = getMoveDirection()
		if not moveDir then return end

		local now = tick()

		if now - lastJumpTime <= maxJumpInterval then
			dashCount = dashCount + 1
		else
			dashCount = 1
		end
		lastJumpTime = now

		if dashCount > maxDashCount then return end

		local animId
		local lookVector = hrp.CFrame.LookVector
		local rightVector = hrp.CFrame.RightVector

		local dotBack = moveDir:Dot(-lookVector)
		local dotRight = moveDir:Dot(rightVector)
		local dotLeft = moveDir:Dot(-rightVector)

		if dotBack > 0.7 then
			animId = Animations.Back
		elseif dotRight > 0.7 then
			animId = Animations.Right
		elseif dotLeft > 0.7 then
			animId = Animations.Left
		else
			animId = "rbxassetid://15998569771"
		end

		local anim = Instance.new("Animation")
		anim.AnimationId = animId
		local track = humanoid:LoadAnimation(anim)
		track:Play()

		rs:WaitForChild("Remotes"):WaitForChild("Effects"):WaitForChild("JumpDashFX"):FireServer(char, "F")

		local bv = Instance.new("BodyVelocity")
		bv.MaxForce = Vector3.new(1e6, 1e5, 1e6)
		bv.Velocity = moveDir * DashPower + Vector3.new(0, 10, 0)
		bv.P = 1250
		bv.Parent = hrp

		task.delay(0.3, function()
			bv:Destroy()
		end)

		if dashCount == maxDashCount then
			cooldown = true
			task.delay(cooldownTime, function()
				cooldown = false
				dashCount = 0
			end)
		end
	end)
end

if player.Character then
	setupCharacter(player.Character)
end
player.CharacterAdded:Connect(setupCharacter)
