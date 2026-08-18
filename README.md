```lua
-- ServerScript (ServerScriptService)

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local remote = Instance.new("RemoteEvent")
remote.Name = "HarvesterHit"
remote.Parent = ReplicatedStorage

local sizeRemote = Instance.new("RemoteEvent")
sizeRemote.Name = "HarvesterSize"
sizeRemote.Parent = ReplicatedStorage

remote.OnServerEvent:Connect(function(player, targetHumanoid)
	if typeof(targetHumanoid) ~= "Instance" or not targetHumanoid:IsA("Humanoid") then return end
	if targetHumanoid.Health <= 0 then return end

	local char = player.Character
	if not char then return end

	local root = char:FindFirstChild("HumanoidRootPart")
	local targetRoot = targetHumanoid.Parent and targetHumanoid.Parent:FindFirstChild("HumanoidRootPart")
	if not root or not targetRoot then return end
	if (root.Position - targetRoot.Position).Magnitude > 500 then return end

	targetHumanoid.Health = 0
end)

sizeRemote.OnServerEvent:Connect(function(player, isBig)
	local char = player.Character
	if not char then return end

	local tool = char:FindFirstChild("Harvester") or player.Backpack:FindFirstChild("Harvester")
	if not tool then return end

	local handle = tool:FindFirstChild("Handle")
	if not handle then return end

	if isBig then
		handle.Size = Vector3.new(4.5, 1.4, 5.5)
	else
		handle.Size = Vector3.new(0.9, 0.28, 1.15)
	end
end)

Players.PlayerAdded:Connect(function(player)
	player.CharacterAdded:Connect(function(char)
		task.wait(1)

		local tool = Instance.new("Tool")
		tool.Name = "Harvester"
		tool.RequiresHandle = true
		tool.CanBeDropped = false
		tool.Grip = CFrame.new(0, -0.1, -0.7) * CFrame.Angles(math.rad(-5), math.rad(180), 0)

		local h = Instance.new("MeshPart")
		h.Name = "Handle"
		h.Size = Vector3.new(0.9, 0.28, 1.15)
		h.Material = Enum.Material.Metal
		h.CanCollide = false
		h.MeshId = "rbxassetid://7775027413"
		h.TextureID = "rbxassetid://7775245551"
		h.Parent = tool

		local a = Instance.new("Part")
		a.Size = Vector3.new(0.12, 0.12, 0.4)
		a.Color = Color3.fromRGB(0, 255, 80)
		a.Material = Enum.Material.Neon
		a.CanCollide = false
		a.Parent = tool

		local w = Instance.new("Weld")
		w.Part0 = h
		w.Part1 = a
		w.C0 = CFrame.new(0, 0.18, 0.1)
		w.Parent = h

		tool.Parent = player.Backpack
	end)
end)
```

```lua
-- LocalScript (StarterPlayerScripts ou dentro do Tool)

local P = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local D = game:GetService("Debris")
local LP = P.LocalPlayer
local M = LP:GetMouse()

local ammo, rel, can, eq = 1, false, true, false
local MAX, RELOAD, RANGE, CD = 1, 1.35, 450, 0.12
local isBig = false

local remote = game:GetService("ReplicatedStorage"):WaitForChild("HarvesterHit")
local sizeRemote = game:GetService("ReplicatedStorage"):WaitForChild("HarvesterSize")

local function reload(tool)
	if rel or ammo >= MAX then return end
	rel = true
	task.delay(RELOAD, function()
		ammo = MAX
		rel = false
	end)
end

local function shoot(tool)
	if not eq or not can or rel or ammo <= 0 then
		if ammo <= 0 and not rel then reload(tool) end
		return
	end
	can = false
	ammo = ammo - 1

	local cam = workspace.CurrentCamera
	local ori = cam.CFrame.Position
	local dir = (M.Hit.Position - ori).Unit * RANGE
	local rp = RaycastParams.new()
	rp.FilterDescendantsInstances = {LP.Character}
	rp.FilterType = Enum.RaycastFilterType.Exclude
	local res = workspace:Raycast(ori, dir, rp)

	local sp = ori + cam.CFrame.LookVector * 2.5
	local ep = res and res.Position or ori + dir

	local b = Instance.new("Part")
	b.Anchored = true
	b.CanCollide = false
	b.Material = Enum.Material.Neon
	b.Color = Color3.fromRGB(0, 255, 80)
	b.Transparency = 0.15
	b.Size = Vector3.new(0.1, 0.1, (sp - ep).Magnitude)
	b.CFrame = CFrame.new(sp, ep) * CFrame.new(0, 0, -b.Size.Z / 2)
	b.Parent = workspace
	D:AddItem(b, 0.13)

	if res and res.Instance then
		local m = res.Instance:FindFirstAncestorOfClass("Model")
		if m then
			local hum = m:FindFirstChildOfClass("Humanoid")
			if hum and hum.Health > 0 then
				remote:FireServer(hum)

				local fx = Instance.new("Part")
				fx.Size = Vector3.new(0.7, 0.7, 0.7)
				fx.Color = Color3.fromRGB(0, 255, 90)
				fx.Material = Enum.Material.Neon
				fx.Anchored = true
				fx.CanCollide = false
				fx.Position = res.Position
				fx.Parent = workspace
				D:AddItem(fx, 0.25)
			end
		end
	end

	task.delay(CD, function() can = true end)
	if ammo <= 0 then reload(tool) end
end

local function onEquipped(tool)
	eq = true
	tool.Activated:Connect(function()
		shoot(tool)
	end)
end

LP.Backpack.ChildAdded:Connect(function(child)
	if child.Name == "Harvester" and child:IsA("Tool") then
		child.Equipped:Connect(function()
			onEquipped(child)
		end)
		child.Unequipped:Connect(function()
			eq = false
		end)
	end
end)

UIS.InputBegan:Connect(function(input, gpe)
	if gpe then return end
	if input.KeyCode == Enum.KeyCode.G then
		isBig = not isBig
		sizeRemote:FireServer(isBig)
	end
end)
```
