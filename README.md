-- Harvester Verde Fake (funcional)
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local Debris = game:GetService("Debris")

local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()

local MAX_AMMO = 1
local RELOAD_TIME = 1.35
local DAMAGE = 40
local RANGE = 450
local COOLDOWN = 0.12

local ammo = MAX_AMMO
local reloading = false
local canShoot = true

local function createHarvester()
	local tool = Instance.new("Tool")
	tool.Name = "Harvester"
	tool.RequiresHandle = true
	tool.CanBeDropped = false
	tool.Grip = CFrame.new(0, -0.4, -0.8) * CFrame.Angles(math.rad(-10), 0, 0)

	-- Handle
	local handle = Instance.new("Part")
	handle.Name = "Handle"
	handle.Size = Vector3.new(0.35, 0.35, 1.8)
	handle.Color = Color3.fromRGB(35, 35, 40)
	handle.Material = Enum.Material.Metal
	handle.Parent = tool

	-- Corpo
	local body = Instance.new("Part")
	body.Name = "Body"
	body.Size = Vector3.new(0.5, 0.4, 1.4)
	body.Color = Color3.fromRGB(25, 25, 30)
	body.Material = Enum.Material.Metal
	body.CanCollide = false
	body.Parent = tool

	local weldBody = Instance.new("Weld")
	weldBody.Part0 = handle
	weldBody.Part1 = body
	weldBody.C0 = CFrame.new(0, 0.1, -0.3)
	weldBody.Parent = handle

	-- Arco superior
	local topBow = Instance.new("Part")
	topBow.Size = Vector3.new(2.1, 0.12, 0.12)
	topBow.Color = Color3.fromRGB(20, 20, 25)
	topBow.Material = Enum.Material.Metal
	topBow.CanCollide = false
	topBow.Parent = tool

	local weldTop = Instance.new("Weld")
	weldTop.Part0 = body
	weldTop.Part1 = topBow
	weldTop.C0 = CFrame.new(0, 0.35, -0.5) * CFrame.Angles(0, 0, math.rad(12))
	weldTop.Parent = body

	-- Arco inferior
	local bottomBow = Instance.new("Part")
	bottomBow.Size = Vector3.new(2.1, 0.12, 0.12)
	bottomBow.Color = Color3.fromRGB(20, 20, 25)
	bottomBow.Material = Enum.Material.Metal
	bottomBow.CanCollide = false
	bottomBow.Parent = tool

	local weldBottom = Instance.new("Weld")
	weldBottom.Part0 = body
	weldBottom.Part1 = bottomBow
	weldBottom.C0 = CFrame.new(0, -0.25, -0.5) * CFrame.Angles(0, 0, math.rad(-12))
	weldBottom.Parent = body

	-- Corda
	local stringPart = Instance.new("Part")
	stringPart.Size = Vector3.new(0.05, 1.1, 0.05)
	stringPart.Color = Color3.fromRGB(180, 180, 180)
	stringPart.Material = Enum.Material.SmoothPlastic
	stringPart.CanCollide = false
	stringPart.Parent = tool

	local weldString = Instance.new("Weld")
	weldString.Part0 = body
	weldString.Part1 = stringPart
	weldString.C0 = CFrame.new(0, 0.05, -1.1)
	weldString.Parent = body

	-- Detalhe verde neon (estilo Harvester)
	local accent = Instance.new("Part")
	accent.Size = Vector3.new(0.15, 0.15, 0.6)
	accent.Color = Color3.fromRGB(0, 220, 70)
	accent.Material = Enum.Material.Neon
	accent.CanCollide = false
	accent.Parent = tool

	local weldAccent = Instance.new("Weld")
	weldAccent.Part0 = body
	weldAccent.Part1 = accent
	weldAccent.C0 = CFrame.new(0, 0.28, 0.1)
	weldAccent.Parent = body

	-- Sons
	local shootSound = Instance.new("Sound")
	shootSound.SoundId = "rbxassetid://138083963"
	shootSound.Volume = 1.6
	shootSound.PlaybackSpeed = 1.1
	shootSound.Parent = handle

	local reloadSound = Instance.new("Sound")
	reloadSound.SoundId = "rbxassetid://269729542"
	reloadSound.Volume = 1.3
	reloadSound.Parent = handle

	tool.Parent = LocalPlayer.Backpack
	return tool, shootSound, reloadSound
end

local tool, shootSound, reloadSound = createHarvester()

local function shoot()
	if not canShoot or reloading or ammo <= 0 then
		if ammo <= 0 and not reloading then
			reload()
		end
		return
	end

	canShoot = false
	ammo -= 1
	shootSound:Play()

	local camera = workspace.CurrentCamera
	local origin = camera.CFrame.Position
	local direction = (Mouse.Hit.Position - origin).Unit * RANGE

	local rayParams = RaycastParams.new()
	rayParams.FilterDescendantsInstances = {LocalPlayer.Character}
	rayParams.FilterType = Enum.RaycastFilterType.Exclude

	local result = workspace:Raycast(origin, direction, rayParams)

	-- Rastro verde
	local startPos = origin + camera.CFrame.LookVector * 2.5
	local endPos = result and result.Position or (origin + direction)

	local beam = Instance.new("Part")
	beam.Anchored = true
	beam.CanCollide = false
	beam.Material = Enum.Material.Neon
	beam.Color = Color3.fromRGB(0, 255, 80)
	beam.Transparency = 0.25
	beam.Size = Vector3.new(0.1, 0.1, (startPos - endPos).Magnitude)
	beam.CFrame = CFrame.new(startPos, endPos) * CFrame.new(0, 0, -beam.Size.Z/2)
	beam.Parent = workspace
	Debris:AddItem(beam, 0.1)

	if result and result.Instance then
		local model = result.Instance:FindFirstAncestorOfClass("Model")
		if model then
			local humanoid = model:FindFirstChildOfClass("Humanoid")
			if humanoid and humanoid.Health > 0 then
				humanoid:TakeDamage(DAMAGE)

				local hitFx = Instance.new("Part")
				hitFx.Size = Vector3.new(0.7, 0.7, 0.7)
				hitFx.Color = Color3.fromRGB(0, 255, 90)
				hitFx.Material = Enum.Material.Neon
				hitFx.Anchored = true
				hitFx.CanCollide = false
				hitFx.Position = result.Position
				hitFx.Parent = workspace
				Debris:AddItem(hitFx, 0.2)
			end
		end
	end

	task.delay(COOLDOWN, function()
		canShoot = true
	end)

	if ammo <= 0 then
		reload()
	end
end

function reload()
	if reloading or ammo >= MAX_AMMO then return end
	reloading = true
	reloadSound:Play()
	print("Recarregando Harvester...")

	task.delay(RELOAD_TIME, function()
		ammo = MAX_AMMO
		reloading = false
		print("Harvester pronta!")
	end)
end

tool.Activated:Connect(shoot)

UserInputService.InputBegan:Connect(function(input, gpe)
	if gpe then return end
	if input.KeyCode == Enum.KeyCode.R then
		reload()
	end
end)

print("Harvester Verde Fake criada! Equipe e atire | R = recarregar")
