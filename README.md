local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local ADMINS = {12345678}
local TELEPORT_DISTANCE = 6
local RAISE_UP = 3

local event = ReplicatedStorage:FindFirstChild("AdminFreezeTeleport")
if not event then
    event = Instance.new("RemoteEvent")
    event.Name = "AdminFreezeTeleport"
    event.Parent = ReplicatedStorage
end

local enemies = {
	workspace.Enemies["Lv1 Crab"],
	workspace.Enemies["Lv11 Boar"],
	workspace.Enemies["Lv12 Boar"],
	workspace.Enemies["Lv12 Thug"],
	workspace.Enemies["Lv13 Boar"],
	workspace.Enemies["Lv13 Bandit"],
	workspace.Enemies["Lv14 Bandit"],
	workspace.Enemies["Lv14 Boar"],
	workspace.Enemies["Lv15 Bandit"],
	workspace.Enemies["Lv15 Boar"],
	workspace.Enemies["Lv15 Thug"],
	workspace.Enemies["Lv16 Boar"],
	workspace.Enemies["Lv16 Thug"],
	workspace.Enemies["Lv17 Thug"],
	workspace.Enemies["Lv186 Cave Demon"],
	workspace.Enemies["Lv188 Cave Demon"],
	workspace.Enemies["Lv19 Thief"],
	workspace.Enemies["Lv198 Cave Demon"],
	workspace.Enemies["Lv2 Angry Bob"],
	workspace.Enemies["Lv20 Thief"],
	workspace.Enemies["Lv200 Vokun"],
	workspace.Enemies["Lv21 Thief"],
	workspace.Enemies["Lv219 Cave Demon"],
	workspace.Enemies["Lv22 Angry Bobby"],
	workspace.Enemies["Lv22 Thief"],
	workspace.Enemies["Lv22 Thug"],
	workspace.Enemies["Lv23 Thug"],
	workspace.Enemies["Lv24 Angry Bobbi"],
	workspace.Enemies["Lv24 Fred"],
	workspace.Enemies["Lv24 Thug"],
	workspace.Enemies["Lv25 Thug"],
	workspace.Enemies["Lv26 Thug"],
	workspace.Enemies["Lv28 Fredde"],
	workspace.Enemies["Lv28 Freyd"],
	workspace.Enemies["Lv28 Friedrich"],
	workspace.Enemies["Lv29 Angry Bobber"],
	workspace.Enemies["Lv29 Frued"],
	workspace.Enemies["Lv3 Crab"],
	workspace.Enemies["Lv30 Thief"],
	workspace.Enemies["Lv30 Thug"],
	workspace.Enemies["Lv31 Thief"],
	workspace.Enemies["Lv32 Fredric"],
	workspace.Enemies["Lv32 Thief"],
	workspace.Enemies["Lv34 Freddi"],
	workspace.Enemies["Lv35 Angry Bobb"],
	workspace.Enemies["Lv4 Boar"],
	workspace.Enemies["Lv4 Crab"],
	workspace.Enemies["Lv4 Freddy"],
	workspace.Enemies["Lv40 Cave Demon"],
	workspace.Enemies["Lv40 Thug"],
	workspace.Enemies["Lv5 Crab"],
	workspace.Enemies["Lv6 Crab"],
	workspace.Enemies["Lv7 Crab"],
	workspace.Enemies["Lv9 Bandit"],
}

local originalStates = {}

local function isAdmin(userId)
	for _, id in ipairs(ADMINS) do
		if id == userId then return true end
	end
	return false
end

local function tryFindHumanoid(model)
	if not model then return nil end
	if model:IsA("Model") then
		return model:FindFirstChildOfClass("Humanoid")
	end
	return nil
end

local function tryFindPrimaryPart(model)
	if not model or not model:IsA("Model") then return nil end
	if model.PrimaryPart then return model.PrimaryPart end
	for _, v in ipairs(model:GetDescendants()) do
		if v:IsA("BasePart") then return v end
	end
	return nil
end

local function freezeEnemy(e)
	if not e then return end
	local state = {
		humanoid = nil,
		prevWalkSpeed = nil,
		prevJumpPower = nil,
		prevPlatformStand = nil,
		anchoredParts = {},
		disabledScripts = {},
	}
	local hum = tryFindHumanoid(e)
	if hum then
		state.humanoid = hum
		state.prevWalkSpeed = hum.WalkSpeed
		state.prevJumpPower = hum.JumpPower or 0
		state.prevPlatformStand = hum.PlatformStand
		hum.WalkSpeed = 0
		if hum.JumpPower ~= nil then hum.JumpPower = 0 end
		hum.PlatformStand = true
		pcall(function() hum:ChangeState(Enum.HumanoidStateType.Physics) end)
	end
	for _, child in ipairs(e:GetDescendants()) do
		if child:IsA("Script") or child:IsA("LocalScript") then
			if child.Disabled == false then
				table.insert(state.disabledScripts, child)
				child.Disabled = true
			end
		end
		if child:IsA("BasePart") and not child.Anchored then
			child.Anchored = true
			table.insert(state.anchoredParts, child)
		end
	end
	originalStates[e] = state
end

local function restoreEnemy(e)
	local state = originalStates[e]
	if not state then return end
	if state.humanoid then
		local hum = state.humanoid
		pcall(function()
			if state.prevWalkSpeed then hum.WalkSpeed = state.prevWalkSpeed end
			if state.prevJumpPower ~= nil and hum.JumpPower ~= nil then hum.JumpPower = state.prevJumpPower end
			if state.prevPlatformStand ~= nil then hum.PlatformStand = state.prevPlatformStand end
		end)
	end
	for _, s in ipairs(state.disabledScripts or {}) do
		pcall(function() s.Disabled = false end)
	end
	for _, p in ipairs(state.anchoredParts or {}) do
		pcall(function() p.Anchored = false end)
	end
	originalStates[e] = nil
end

local function freezeAll()
	for _, e in ipairs(enemies) do
		if e and e.Parent then
			freezeEnemy(e)
		end
	end
end

local function restoreAll()
	for e, _ in pairs(originalStates) do
		if e and e.Parent then
			restoreEnemy(e)
		end
	end
	originalStates = {}
end

local function teleportEnemiesInFrontOfPlayer(player, delayBetween)
	local char = player.Character
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	if not hrp then return end
	for _, e in ipairs(enemies) do
		if e and e.Parent then
			local primary = tryFindPrimaryPart(e)
			if primary then
				local forward = hrp.CFrame.LookVector
				local targetPos = hrp.Position + forward.Unit * TELEPORT_DISTANCE + Vector3.new(0, RAISE_UP, 0)
				pcall(function()
					if e:IsA("Model") and e.PrimaryPart then
						e:SetPrimaryPartCFrame(CFrame.new(targetPos))
					elseif e:IsA("Model") and not e.PrimaryPart then
						primary.CFrame = CFrame.new(targetPos)
					elseif primary:IsA("BasePart") then
						primary.CFrame = CFrame.new(targetPos)
					end
				end)
			end
			wait(delayBetween or 0.5)
		end
	end
end

event.OnServerEvent:Connect(function(player, action, delayBetween)
	if #ADMINS > 0 and not isAdmin(player.UserId) then return end
	action = action or "freeze_and_teleport"
	if action == "freeze_and_teleport" then
		freezeAll()
		teleportEnemiesInFrontOfPlayer(player, delayBetween or 0.5)
	elseif action == "freeze_only" then
		freezeAll()
	elseif action == "teleport_only" then
		teleportEnemiesInFrontOfPlayer(player, delayBetween or 0.5)
	elseif action == "restore" then
		restoreAll()
	end
end)

workspace.Enemies.DescendantRemoving:Connect(function(desc)
	if originalStates[desc] then originalStates[desc] = nil end
end)
