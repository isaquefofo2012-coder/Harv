local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Debris = game:GetService("Debris")

local remote = ReplicatedStorage:FindFirstChild("HarvesterFire")

if not remote then
	remote = Instance.new("RemoteEvent")
	remote.Name = "HarvesterFire"
	remote.Parent = ReplicatedStorage
end

local DAMAGE = 40
local RANGE = 450
local COOLDOWN = 0.12

local lastShot = {}

remote.OnServerEvent:Connect(function(player, targetPosition)
	if typeof(targetPosition) ~= "Vector3" then
		return
	end

	local character = player.Character
	if not character then
		return
	end

	local humanoid = character:FindFirstChildOfClass("Humanoid")
	local head = character:FindFirstChild("Head")

	if not humanoid or humanoid.Health <= 0 or not head then
		return
	end

	local now = os.clock()

	if lastShot[player] and now - lastShot[player] < COOLDOWN then
		return
	end

	lastShot[player] = now

	local origin = head.Position
	local offset = targetPosition - origin

	if offset.Magnitude <= 0 then
		return
	end

	local direction = offset.Unit * math.min(offset.Magnitude, RANGE)

	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = {character}

	local result = workspace:Raycast(origin, direction, params)

	if not result then
		return
	end

	local model = result.Instance:FindFirstAncestorOfClass("Model")

	if not model then
		return
	end

	local targetHumanoid = model:FindFirstChildOfClass("Humanoid")

	if targetHumanoid and targetHumanoid.Health > 0 then
		targetHumanoid:TakeDamage(DAMAGE)
	end
end)

game.Players.PlayerRemoving:Connect(function(player)
	lastShot[player] = nil
end)
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local Debris = game:GetService("Debris")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local mouse = player:GetMouse()

local remote = ReplicatedStorage:WaitForChild("HarvesterFire")

local MAX_AMMO = 1
local RELOAD_TIME = 1.35
local COOLDOWN = 0.12
local RANGE = 450

local ammo = MAX_AMMO
local reloading = false
local canShoot = true

local function createHarvester()
	local tool = Instance.new("Tool")
	tool.Name = "Harvester"
	tool.RequiresHandle = true
	tool.CanBeDropped = false
	tool.Grip = CFrame.new(0, -0.4, -0.8)
		* CFrame.Angles(math.rad(-10), 0, 0)

	local handle = Instance.new("Part")
	handle.Name = "Handle"
	handle.Size = Vector3.new(0.35, 0.35, 1.8)
	handle.Color = Color3.fromRGB(35, 35, 40)
	handle.Material = Enum.Material.Metal
	handle.CanCollide = false
	handle.Parent = tool

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

	local topBow = Instance.new("Part")
	topBow.Name = "TopBow"
	topBow.Size = Vector3.new(2.1, 0.12, 0.12)
	topBow.Color = Color3.fromRGB(20, 20, 25)
	topBow.Material = Enum.Material.Metal
	topBow.CanCollide = false
	topBow.Parent = tool

	local weldTop = Instance.new("Weld")
	weldTop.Part0 = body
	weldTop.Part1 = topBow
	weldTop.C0 = CFrame.new(0, 0.35, -0.5)
		* CFrame.Angles(0, 0, math.rad(12))
	weldTop.Parent = body

	local bottomBow = Instance.new("Part")
	bottomBow.Name = "BottomBow"
	bottomBow.Size = Vector3.new(2.1, 0.12, 0.12)
	bottomBow.Color = Color3.fromRGB(20, 20, 25)
	bottomBow.Material = Enum.Material.Metal
	bottomBow.CanCollide = false
	bottomBow.Parent = tool

	local weldBottom = Instance.new("Weld")
	weldBottom.Part0 = body
	weldBottom.Part1 = bottomBow
	weldBottom.C0 = CFrame.new(0, -0.25, -0.5)
		* CFrame.Angles(0, 0, math.rad(-12))
	weldBottom.Parent = body

	local stringPart = Instance.new("Part")
	stringPart.Name = "String"
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

	local accent = Instance.new("Part")
	accent.Name = "GreenAccent"
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

	local shootSound = Instance.new("Sound")
	shootSound.Name = "ShootSound"
	shootSound.SoundId = "rbxassetid://138083963"
	shootSound.Volume = 1.6
	shootSound.PlaybackSpeed = 1.1
	shootSound.Parent = handle

	local reloadSound = Instance.new("Sound")
	reloadSound.Name = "ReloadSound"
	reloadSound.SoundId = "rbxassetid://269729542"
	reloadSound.Volume = 1.3
	reloadSound.Parent = handle

	return tool, shootSound, reloadSound
end

local tool, shootSound, reloadSound = createHarvester()
tool.Parent = player:WaitForChild("Backpack")

local function reload()
	if reloading or ammo >= MAX_AMMO then
		return
	end

	reloading = true

	if reloadSound then
		reloadSound:Play()
	end

	print("Recarregando Harvester...")

	task.delay(RELOAD_TIME, function()
		if not tool or not tool.Parent then
			reloading = false
			return
		end

		ammo = MAX_AMMO
		reloading = false

		print("Harvester pronta!")
	end)
end

local function createBeam(startPos, endPos)
	local distance = (startPos - endPos).Magnitude

	local beam = Instance.new("Part")
	beam.Name = "HarvesterBeam"
	beam.Anchored = true
	beam.CanCollide = false
	beam.Material = Enum.Material.Neon
	beam.Color = Color3.fromRGB(0, 255, 80)
	beam.Transparency = 0.25
	beam.Size = Vector3.new(0.1, 0.1, distance)

	beam.CFrame =
		CFrame.new(startPos, endPos)
		* CFrame.new(0, 0, -distance / 2)

	beam.Parent = workspace

	Debris:AddItem(beam, 0.1)
end

local function createHitEffect(position)
	local hitFx = Instance.new("Part")

	hitFx.Name = "HarvesterHitFX"
	hitFx.Size = Vector3.new(0.7, 0.7, 0.7)
	hitFx.Color = Color3.fromRGB(0, 255, 90)
	hitFx.Material = Enum.Material.Neon
	hitFx.Anchored = true
	hitFx.CanCollide = false
	hitFx.Position = position
	hitFx.Parent = workspace

	Debris:AddItem(hitFx, 0.2)
end

local function shoot()
	if not canShoot or reloading then
		return
	end

	if ammo <= 0 then
		reload()
		return
	end

	canShoot = false
	ammo -= 1

	if shootSound then
		shootSound:Play()
	end

	local camera = workspace.CurrentCamera

	if not camera then
		canShoot = true
		return
	end

	local origin = camera.CFrame.Position
	local target = mouse.Hit.Position

	local offset = target - origin

	if offset.Magnitude <= 0 then
		canShoot = true
		return
	end

	local direction = offset.Unit * math.min(offset.Magnitude, RANGE)

	local rayParams = RaycastParams.new()
	rayParams.FilterType = Enum.RaycastFilterType.Exclude
	rayParams.FilterDescendantsInstances = {
		player.Character
	}

	local result = workspace:Raycast(
		origin,
		direction,
		rayParams
	)

	local endPos

	if result then
		endPos = result.Position
	else
		endPos = origin + direction
	end

	local startPos =
		origin + camera.CFrame.LookVector * 2.5

	createBeam(startPos, endPos)

	if result then
		createHitEffect(result.Position)
	end

	-- Agora o cliente apenas solicita o disparo.
	-- O servidor é quem valida e aplica o dano.
	remote:FireServer(target)

	task.delay(COOLDOWN, function()
		canShoot = true
	end)

	if ammo <= 0 then
		reload()
	end
end

tool.Activated:Connect(shoot)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then
		return
	end

	if input.KeyCode == Enum.KeyCode.R then
		reload()
	end
end)

print("Harvester Verde carregada!")
print("Clique = atirar | R = recarregar")
