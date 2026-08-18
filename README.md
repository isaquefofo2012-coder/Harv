-- Harvester Verde Fake (modelo oficial + tiro por toque)
-- Mesh & Texture oficiais do MM2

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local Debris = game:GetService("Debris")
local RunService = game:GetService("RunService")

local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()

-- Configurações
local MAX_AMMO = 1
local RELOAD_TIME = 1.35
local DAMAGE = 40
local RANGE = 450
local COOLDOWN = 0.12

local ammo = MAX_AMMO
local reloading = false
local canShoot = true
local toolEquipped = false

local function createHarvester()
	local tool = Instance.new("Tool")
	tool.Name = "Harvester"
	tool.RequiresHandle = true
	tool.CanBeDropped = false
	-- Grip ajustado para o modelo oficial
	tool.Grip = CFrame.new(0, -0.15, -0.9) * CFrame.Angles(math.rad(-8), math.rad(180), 0)

	-- Handle com o mesh oficial da Harvester
	local handle = Instance.new("MeshPart")
	handle.Name = "Handle"
	handle.Size = Vector3.new(2.25, 0.66, 2.88) -- tamanho próximo do original
	handle.Color = Color3.fromRGB(25, 25, 30)
	handle.Material = Enum.Material.Metal
	handle.CanCollide = false
	handle.CastShadow = true
	handle.MeshId = "rbxassetid://7775027413"
	handle.TextureID = "rbxassetid://7775245551"
	handle.Parent = tool

	-- Accent verde neon (detalhe extra para a versão "Verde")
	local accent = Instance.new("Part")
	accent.Name = "NeonAccent"
	accent.Size = Vector3.new(0.18, 0.18, 0.55)
	accent.Color = Color3.fromRGB(0, 255, 80)
	accent.Material = Enum.Material.Neon
	accent.CanCollide = false
	accent.Parent = tool

	local weldAccent = Instance.new("Weld")
	weldAccent.Part0 = handle
	weldAccent.Part1 = accent
	weldAccent.C0 = CFrame.new(0, 0.32, 0.15)
	weldAccent.Parent = handle

	-- Sons
	local shootSound = Instance.new("Sound")
	shootSound.Name = "Shoot"
	shootSound.SoundId = "rbxassetid://3319653103"
	shootSound.Volume = 1.8
	shootSound.PlaybackSpeed = 1.15
	shootSound.Parent = handle

	local reloadSound = Instance.new("Sound")
	reloadSound.Name = "Reload"
	reloadSound.SoundId = "rbxassetid://5487661538"
	reloadSound.Volume = 1.4
	reloadSound.Parent = handle

	tool.Parent = LocalPlayer.Backpack
	return tool, shootSound, reloadSound
end

local tool, shootSound, reloadSound = createHarvester()

local function playSound(original)
	local s = original:Clone()
	s.Parent = tool.Handle
	s:Play()
	Debris:AddItem(s, 3)
end

local function shoot()
	if not toolEquipped or not canShoot or reloading or ammo <= 0 then
		if ammo <= 0 and not reloading then
			reload()
		end
		return
	end

	canShoot = false
	ammo -= 1
	playSound(shootSound)

	local camera = workspace.CurrentCamera
	local origin = camera.CFrame.Position
	local direction = (Mouse.Hit.Position - origin).Unit * RANGE

	local rayParams = RaycastParams.new()
	rayParams.FilterDescendantsInstances = {LocalPlayer.Character}
	rayParams.FilterType = Enum.RaycastFilterType.Exclude

	local result = workspace:Raycast(origin, direction, rayParams)

	-- Rastro verde neon
	local startPos = origin + camera.CFrame.LookVector * 2.8
	local endPos = result and result.Position or (origin + direction)

	local beam = Instance.new("Part")
	beam.Anchored = true
	beam.CanCollide = false
	beam.Material = Enum.Material.Neon
	beam.Color = Color3.fromRGB(0, 255, 80)
	beam.Transparency = 0.15
	beam.Size = Vector3.new(0.11, 0.11, (startPos - endPos).Magnitude)
	beam.CFrame = CFrame.new(startPos, endPos) * CFrame.new(0, 0, -beam.Size.Z / 2)
	beam.Parent = workspace
	Debris:AddItem(beam, 0.13)

	if result and result.Instance then
		local model = result.Instance:FindFirstAncestorOfClass("Model")
		if model then
			local humanoid = model:FindFirstChildOfClass("Humanoid")
			if humanoid and humanoid.Health > 0 then
				-- Dano client-side (funciona bem em NPCs / private)
				humanoid:TakeDamage(DAMAGE)

				-- Efeito de hit verde
				local hitFx = Instance.new("Part")
				hitFx.Size = Vector3.new(0.85, 0.85, 0.85)
				hitFx.Color = Color3.fromRGB(0, 255, 90)
				hitFx.Material = Enum.Material.Neon
				hitFx.Anchored = true
				hitFx.CanCollide = false
				hitFx.Position = result.Position
				hitFx.Parent = workspace
				Debris:AddItem(hitFx, 0.28)

				local attachment = Instance.new("Attachment")
				attachment.Parent = hitFx

				local particles = Instance.new("ParticleEmitter")
				particles.Color = ColorSequence.new(Color3.fromRGB(0, 255, 100))
				particles.Size = NumberSequence.new({
					NumberSequenceKeypoint.new(0, 0.45),
					NumberSequenceKeypoint.new(1, 0)
				})
				particles.Lifetime = NumberRange.new(0.25, 0.4)
				particles.Speed = NumberRange.new(6, 12)
				particles.Rate = 0
				particles.Parent = attachment
				particles:Emit(14)
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
	playSound(reloadSound)
	print("Recarregando Harvester...")

	task.delay(RELOAD_TIME, function()
		ammo = MAX_AMMO
		reloading = false
		print("Harvester pronta!")
	end)
end

-- Equip / Unequip
tool.Equipped:Connect(function()
	toolEquipped = true
end)

tool.Unequipped:Connect(function()
	toolEquipped = false
end)

-- Tiro por clique / toque + tecla R
UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end
	if not toolEquipped then return end

	if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
		shoot()
	end

	if input.KeyCode == Enum.KeyCode.R then
		reload()
	end
end)

print("Harvester Verde Fake (modelo oficial) pronta!")
print("Equip → clique/toque para atirar | R = recarregar")
