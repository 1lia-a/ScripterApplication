--// Services
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService      = game:GetService("TweenService")
local Debris            = game:GetService("Debris")

--// Remote
local AbilityFunction: RemoteFunction = ReplicatedStorage.AbilityFunction

--// Assets
local Assets = script:WaitForChild("Assets")
local Sounds = script:WaitForChild("Sounds")

--// Utils
local function _lerp(a, b, t)
	return a + (b - a) * t
end

local function _quadBazie(t, p0, p1, p2)
	return _lerp(_lerp(p0, p1, t), _lerp(p1, p2, t), t)
end

--// Meta & Abilities

local AbilityMeta = {}
AbilityMeta.__index = AbilityMeta

function AbilityMeta.new(name: string, cooldown: number, action)
	return setmetatable({
		Name = name,
		Cooldown = cooldown,
		LastUsed = 0,
		Action = action,
	}, AbilityMeta)
end

function AbilityMeta:Activate(player, Target: Vector3)
	if tick() - self.LastUsed >= self.Cooldown then
		self.LastUsed = tick()
		self.Action(player, Target)
	else
		warn(self.Name .. " is on cooldown for", player)
	end
end

--\\\\\\\\\\\\\\\\\\\\\\\\

local Abilities = {}
local Ability = {}

function Ability:Dash()
	if self.Character then
		local ABILITY_TIME: number = .5		
		local character: Model = self.Character
		local humanoid: Humanoid = character:FindFirstChildOfClass("Humanoid")
		if humanoid then
			-- SFX

			local newSound: Sound = Sounds.Dash:Clone()
			newSound.Parent = character.HumanoidRootPart
			newSound:Play()
			newSound.PlayOnRemove = true

			-- VFX
			
			local VFX_Table = {}

			for i, emitter in pairs(Assets.DashVFX:GetChildren()) do
				if emitter:IsA("ParticleEmitter") then
					VFX_Table[i] = emitter:Clone()
					VFX_Table[i].Parent = character.HumanoidRootPart
					VFX_Table[i].Enabled = true

					Debris:AddItem(VFX_Table[i], ABILITY_TIME + 5)
				end
			end

			for _, object in pairs(character:GetDescendants()) do
				if object:IsA("BasePart") or object:IsA("MeshPart") then
					if object ~= character.HumanoidRootPart then
						object.Transparency = .75
					end
				end
			end

			spawn(function()
				task.wait(ABILITY_TIME)

				for _, object in pairs(character:GetDescendants()) do
					if object:IsA("BasePart") or object:IsA("MeshPart") then
						if object ~= character.HumanoidRootPart then
							object.Transparency = 0
						end
					end
				end

				for i, emitter in pairs(VFX_Table) do
					emitter.Enabled = false
				end
			end)

			-- Ability

			humanoid.WalkSpeed = 56
			task.wait(ABILITY_TIME)
			humanoid.WalkSpeed = 16
		end
	end	
end

function Ability:Explosion(target: Vector3)
	if self.Character then
		local character: Model = self.Character
		local FireArrow: Part = Assets.FireArrow
		local HRMAtts:   Part = Assets.HRMAtts
		local TPart:     Part = Assets.TrajectoryPart
		local HitPart:   Part = Assets.HitPart
		local LIFETIME:  number = 10
		
		-- SFX

		local AttackAtt: Attachment = HRMAtts.Attack:Clone()
		AttackAtt.Parent = character.HumanoidRootPart
		AttackAtt.Flame:Play()
		
		-- VFX
		
		for _, emitter in pairs(AttackAtt:GetChildren()) do
			if emitter:IsA("ParticleEmitter") then
				emitter:Emit(emitter:GetAttribute("EmitCount"))
			end
		end

		local newArrow: Part = FireArrow:Clone()
		newArrow.CFrame = AttackAtt.WorldCFrame
		newArrow.Parent = workspace.Projectiles
		
		-- Calculate the trajectory of the arrow
		
		local Mag = (character.HumanoidRootPart.Position - target).Magnitude
		local TPositions = {
			AttackAtt.WorldCFrame.Position,
			(AttackAtt.WorldCFrame.Position + target) / 2,
			target
		}

		local TParts = {}
		for i = 0, 1, .1  do
			local newTPart = TPart:Clone()
			newTPart.CFrame = CFrame.new(_quadBazie(i, TPositions[1], TPositions[2] + Vector3.new(0, 10, 0), TPositions[3]))
			newTPart.Parent = workspace

			table.insert(TParts, newTPart)

			Debris:AddItem(newTPart, LIFETIME)
		end
		
		-- Launch
		
		for _, TPart in pairs(TParts) do
			local targetCFrame = CFrame.lookAt(newArrow.Position, TPart.Position)
			newArrow.CFrame = targetCFrame

			local Tween = TweenService:Create(newArrow, TweenInfo.new(Mag/1000, Enum.EasingStyle.Linear), {
				Position = TPart.Position
			})
			Tween:Play()
			Tween.Completed:Wait()
		end
		
		-- VFX
		
		for _, emitter in pairs(newArrow.Att:GetChildren()) do
			if emitter:IsA("ParticleEmitter") then
				emitter.Enabled = false
			end
		end
		
		local newHitPart = HitPart:Clone()
		newHitPart.Position = target + Vector3.new(0, .25, 0)
		newHitPart.Parent = workspace
		newHitPart.HitSound:Play()
		
		task.wait(.1)
		
		for _, emitter in pairs(newHitPart:GetDescendants()) do
			if emitter:IsA("ParticleEmitter") then
				emitter:Emit(emitter:GetAttribute("EmitCount"))
			end
		end
		
		Debris:AddItem(newHitPart, LIFETIME)
		Debris:AddItem(newArrow, LIFETIME)
	end
end

function Ability:Shield()
	if self.Character then
		local LIFETIME = 3
		local character: Model = self.Character

		-- SFX

		local newSound: Sound = Sounds.Shield:Clone()
		newSound.Parent = character.HumanoidRootPart
		newSound:Play()
		newSound.PlayOnRemove = true

		-- Ability

		local newShield: Part = Assets.ShieldPart:Clone()
		newShield.CFrame = character.HumanoidRootPart.CFrame
		newShield.Size = Vector3.new(1,1,1)
		newShield.Parent = workspace

		TweenService:Create(newShield, TweenInfo.new(1, Enum.EasingStyle.Bounce), {
			Size = Vector3.new(25,25,25)
		}):Play()

		task.wait(3)

		for _, emitter in pairs(newShield.Core:GetChildren()) do
			if emitter:IsA("ParticleEmitter") then
				emitter.Enabled = false
			end
		end

		TweenService:Create(newShield, TweenInfo.new(.5, Enum.EasingStyle.Bounce), {
			Size = Vector3.new(1,1,1),
			Transparency = 1
		}):Play()

		Debris:AddItem(newShield, 5)
	end
end

--// Registry

Abilities["Dash"] = AbilityMeta.new("Dash", 3, function(player)
	local self = { Character = player.Character }
	setmetatable(self, { __index = Ability })
	self:Dash()
end)

Abilities["Explosion"] = AbilityMeta.new("Explosion", 1, function(player, target: Vector3)
	local self = { Character = player.Character }
	setmetatable(self, { __index = Ability })
	self:Explosion(target)
end)

Abilities["Shield"] = AbilityMeta.new("Shield", 5, function(player)
	local self = { Character = player.Character }
	setmetatable(self, { __index = Ability })
	self:Shield()
end)

--// Remote hook

AbilityFunction.OnServerInvoke = function(player, abilityName: string, target: Vector3)
	local ability = Abilities[abilityName]
	if ability then
		ability:Activate(player, target)
	else
		warn("Ability not found:", abilityName)
	end
end
