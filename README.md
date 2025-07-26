local _ENV = (getgenv or getrenv or getfenv)()
_ENV.OnLevelFarm = false

local TweenService = game:GetService("TweenService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer = Players.LocalPlayer
local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
local HumanoidRootPart = Character:WaitForChild("HumanoidRootPart")

local ServerMove = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("ServerMove")
local NPCFolder = workspace:WaitForChild("NPC"):WaitForChild("Fight")
local ATTACK_RADIUS = 20 -- Raio do Kill Aura

local BodyVelocity
do
	BodyVelocity = Instance.new("BodyVelocity")
	BodyVelocity.Velocity = Vector3.zero
	BodyVelocity.MaxForce = Vector3.new(math.huge * 50, math.huge * 50, math.huge * 50)
	BodyVelocity.P = 1000

	if _ENV.tween_bodyvelocity then
		_ENV.tween_bodyvelocity:Destroy()
	end

	_ENV.tween_bodyvelocity = BodyVelocity

	local CanCollideObjects = {}

	local function AddObjectToBaseParts(Object)
		if Object:IsA("BasePart") and Object.CanCollide then
			table.insert(CanCollideObjects, Object)
		end
	end

	local function RemoveObjectsFromBaseParts(BasePart)
		local index = table.find(CanCollideObjects, BasePart)

		if index then
			table.remove(CanCollideObjects, index)
		end
	end

	local function NewCharacter(Character)
		table.clear(CanCollideObjects)

		for _, Object in Character:GetDescendants() do
			AddObjectToBaseParts(Object)
		end
		Character.DescendantAdded:Connect(AddObjectToBaseParts)
		Character.DescendantRemoving:Connect(RemoveObjectsFromBaseParts)
	end

	table.insert(Connections, Player.CharacterAdded:Connect(NewCharacter))
	task.spawn(NewCharacter, Player.Character)

	local function NoClipOnStepped(Character)
		if _ENV.OnFarm then
			for i = 1, #CanCollideObjects do
				CanCollideObjects[i].CanCollide = false
			end
		elseif Character.PrimaryPart and not Character.PrimaryPart.CanCollide then
			for i = 1, #CanCollideObjects do
				CanCollideObjects[i].CanCollide = true
			end
		end
	end

	local function UpdateVelocityOnStepped(Character)
		local BasePart = Character:FindFirstChild("UpperTorso")
		local Humanoid = Character:FindFirstChild("Humanoid")
		local BodyVelocity = _ENV.tween_bodyvelocity

		if _ENV.OnFarm and BasePart and Humanoid and Humanoid.Health > 0 then
			if BodyVelocity.Parent ~= BasePart then
				BodyVelocity.Parent = BasePart
			end
		elseif BodyVelocity.Parent then
			BodyVelocity.Parent = nil
		end

		if BodyVelocity.Velocity ~= Vector3.zero and (not Humanoid or not Humanoid.SeatPart or not _ENV.OnFarm) then
			BodyVelocity.Velocity = Vector3.zero
		end
	end

	table.insert(
		Connections,
		RunService.Stepped:Connect(function()
			local Character = Player.Character

			if IsAlive(Character) then
				UpdateVelocityOnStepped(Character)
				NoClipOnStepped(Character)
			end
		end)
	)
end

local PlayerTP
do
	local TweenCreator = {}
	do
		TweenCreator.__index = TweenCreator

		local tweens = {}
		local EasingStyle = Enum.EasingStyle.Linear

		function TweenCreator.new(obj, time, prop, value)
			local self = setmetatable({}, TweenCreator)

			self.tween = TweenService:Create(obj, TweenInfo.new(time, EasingStyle), { [prop] = value })
			self.tween:Play()
			self.value = value
			self.object = obj

			if tweens[obj] then
				tweens[obj]:destroy()
			end

			tweens[obj] = self
			return self
		end

		function TweenCreator:destroy()
			self.tween:Pause()
			self.tween:Destroy()

			tweens[self.object] = nil
			setmetatable(self, nil)
		end

		function TweenCreator:stopTween(obj)
			if obj and tweens[obj] then
				tweens[obj]:destroy()
			end
		end
	end

	local function TweenStopped()
		if not BodyVelocity.Parent and IsAlive(Player.Character) then
			TweenCreator:stopTween(Player.Character:FindFirstChild("HumanoidRootPart"))
		end
	end

	local lastCFrame = nil
	local lastTeleport = 0
	local TweenSpeed = 75

	PlayerTP = function(TargetCFrame)
		if not IsAlive(Player.Character) or not Player.Character.PrimaryPart then
			return false
		elseif (tick() - lastTeleport) <= 1 and lastCFrame == TargetCFrame then
			return false
		end

		local Character = Player.Character
		local Humanoid = Character.Humanoid
		local PrimaryPart = Character.PrimaryPart

		if Humanoid.Sit then
			Humanoid.Sit = false
			return
		end

		lastTeleport = tick()
		lastCFrame = TargetCFrame

		local teleportPosition = TargetCFrame.Position
		local Distance = (PrimaryPart.Position - teleportPosition).Magnitude

		if Distance < TweenSpeed then
			PrimaryPart.CFrame = TargetCFrame
			return TweenCreator:stopTween(PrimaryPart)
		end

		TweenCreator.new(PrimaryPart, Distance / TweenSpeed, "CFrame", TargetCFrame)
	end

	table.insert(Connections, BodyVelocity:GetPropertyChangedSignal("Parent"):Connect(TweenStopped))
end

-- Função para obter todos os NPCs próximos
local function getNearbyNPCs(radius)
	local nearby = {}
	for _, npc in ipairs(NPCFolder:GetChildren()) do
		if npc:IsA("Model") and npc:FindFirstChild("HumanoidRootPart") then
			local distance = (npc.HumanoidRootPart.Position - HumanoidRootPart.Position).Magnitude
			if distance <= radius then
				table.insert(nearby, npc)
			end
		end
	end
	return nearby
end

function GetQuestThings(character)
	local lv = ReplicatedStorage:WaitForChild("PlayerData")[LocalPlayer.UserId].Level.Value
	if lv <= 9 then
		_ENV.QuestNpc = workspace:WaitForChild("NPC"):WaitForChild("Talk"):WaitForChild("Woppa"):WaitForChild("Info")
		_ENV.QuestCFrame =
			workspace:WaitForChild("NPC"):WaitForChild("Talk"):WaitForChild("Woppa").HumanoidRootPart.CFrame
		_ENV.NpcToKillFolder = workspace.NPC.Fight.Bandits
		_ENV.NpcToKill = "Bandit"
	elseif lv <= 14 then
		_ENV.QuestNpc = workspace:WaitForChild("NPC"):WaitForChild("Talk"):WaitForChild("Abu"):WaitForChild("Info")
		_ENV.QuestCFrame =
			workspace:WaitForChild("NPC"):WaitForChild("Talk"):WaitForChild("Abu").HumanoidRootPart.CFrame
		_ENV.NpcToKillFolder = workspace.NPC.Fight.Bandits
		_ENV.NpcToKill = "Bandit Leader"
	end
end


function AnbandonQuest()
	game:GetService("ReplicatedStorage"):WaitForChild("Remotes"):WaitForChild("ToServer"):WaitForChild("AbandonQuest")
end

function GetQuest()
	PlayerTP(_ENV.QuestCFrame)
	local args = {
		_G.QuestNpc,
	}
	game:GetService("ReplicatedStorage")
		:WaitForChild("Remotes")
		:WaitForChild("ToServer")
		:WaitForChild("Quest")
		:FireServer(unpack(args))
end

-- Loop principal Kill Aura
function DealDamage(targets)
	if #targets > 0 then
		local args = {
			"CombatV1",
			0.1,
			targets,
			"Combat",
			0.2,
		}
		ServerMove:FireServer(unpack(args))
	end
end

function FarmLevel()
	if _ENV.OnLevelFarm then
		for _, enemy in ipairs(_ENV.NpcToKillFolder:GetChildren()) do
			if enemy:IsA("Model") and enemy:FindFirstChild("HumanoidRootPart") then
				repeat
				PlayerTP(enemy.HumanoidRootPart.CFrame * CFrame.new(0, 5, 0))
				DealDamage(enemy)
				until enemy.Humanoid.Health <= 0
			end
		end
	end
end

local Libary = loadstring(game:HttpGet("https://raw.githubusercontent.com/tlredz/Library/refs/heads/main/V5/Source.lua"))()
local Window = Libary:MakeWindow({ "Vox seas Hub", "Ez levelmax", "rz-VoxSeas.json" })

local MainTab = Window:MakeTab({ "Farm", "Home" })

do
	MainTab:AddSection("Farming")
	MainTab:AddToggle({
		"Farm level",
		false,
		function(Value)
			_ENV.OnLevelFarm = Value
			FarmLevel()
		end,
	})
end
