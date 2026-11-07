-- LocalScript: place in StarterPlayer > StarterPlayerScripts
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local MAX_CLONES = 10000
local FOLLOW_DISTANCE = 4
local REPATH_INTERVAL = 0.25
local FADE_IN_TIME = 0.45
local FADE_OUT_TIME = 0.35
local GLOW_COLOR = Color3.fromRGB(120, 200, 255)

local PANEL_WIDTH = 240
local HEADER_HEIGHT = 40
local PANEL_OPEN_HEIGHT = 230

local COLOR_PANEL = Color3.fromRGB(28, 30, 36)
local COLOR_FIELD = Color3.fromRGB(42, 45, 54)
local COLOR_GREEN = Color3.fromRGB(46, 150, 90)
local COLOR_AMBER = Color3.fromRGB(180, 130, 40)
local COLOR_RED = Color3.fromRGB(170, 60, 60)
local COLOR_BLUE = Color3.fromRGB(60, 110, 180)
local COLOR_TEXT = Color3.fromRGB(240, 240, 240)
local COLOR_MUTED = Color3.fromRGB(170, 175, 185)

local clones = {}
local following = false
local minimized = false
local repathTimer = 0

local function corner(parent, radius)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, radius)
	c.Parent = parent
	return c
end

local function makeButton(parent, text, position, size, color)
	local button = Instance.new("TextButton")
	button.Position = position
	button.Size = size
	button.BackgroundColor3 = color
	button.BorderSizePixel = 0
	button.Text = text
	button.TextColor3 = COLOR_TEXT
	button.TextSize = 15
	button.Font = Enum.Font.GothamBold
	button.Parent = parent
	corner(button, 8)
	return button
end

local gui = Instance.new("ScreenGui")
gui.Name = "CloneFollowUI"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.Parent = playerGui

local panel = Instance.new("Frame")
panel.Name = "Panel"
panel.Size = UDim2.fromOffset(PANEL_WIDTH, PANEL_OPEN_HEIGHT)
panel.Position = UDim2.new(0, 16, 0.5, -115)
panel.BackgroundColor3 = COLOR_PANEL
panel.BorderSizePixel = 0
panel.Parent = gui
corner(panel, 12)

local header = Instance.new("Frame")
header.Name = "Header"
header.Size = UDim2.new(1, 0, 0, HEADER_HEIGHT)
header.BackgroundTransparency = 1
header.Active = true
header.Parent = panel

local title = Instance.new("TextLabel")
title.Position = UDim2.fromOffset(16, 8)
title.Size = UDim2.new(1, -64, 0, 24)
title.BackgroundTransparency = 1
title.Text = "Clone Follow"
title.TextColor3 = COLOR_TEXT
title.TextSize = 20
title.Font = Enum.Font.GothamBold
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = header

local minimizeButton = makeButton(header, "-", UDim2.new(1, -40, 0, 8), UDim2.fromOffset(28, 24), COLOR_FIELD)

local body = Instance.new("Frame")
body.Name = "Body"
body.Position = UDim2.fromOffset(0, HEADER_HEIGHT)
body.Size = UDim2.new(1, 0, 0, PANEL_OPEN_HEIGHT - HEADER_HEIGHT)
body.BackgroundTransparency = 1
body.Parent = panel

local makeCloneButton = makeButton(body, "Make Clone", UDim2.fromOffset(16, 6), UDim2.new(1, -32, 0, 36), COLOR_BLUE)
local followButton = makeButton(body, "Follow Me: Off", UDim2.fromOffset(16, 50), UDim2.new(1, -32, 0, 36), COLOR_GREEN)
local clearButton = makeButton(body, "Clear Clones", UDim2.fromOffset(16, 94), UDim2.new(1, -32, 0, 36), COLOR_RED)

local status = Instance.new("TextLabel")
status.Position = UDim2.fromOffset(16, 138)
status.Size = UDim2.new(1, -32, 0, 36)
status.BackgroundTransparency = 1
status.Text = "0 / " .. MAX_CLONES .. " clones"
status.TextColor3 = COLOR_MUTED
status.TextSize = 13
status.Font = Enum.Font.Gotham
status.TextWrapped = true
status.TextXAlignment = Enum.TextXAlignment.Left
status.TextYAlignment = Enum.TextYAlignment.Top
status.Parent = body

local function updateStatus(message)
	status.Text = message or (#clones .. " / " .. MAX_CLONES .. " clones")
end

local dragging = false
local dragStart
local dragOrigin

header.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = input.Position
		dragOrigin = panel.AbsolutePosition
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				dragging = false
			end
		end)
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if not dragging then
		return
	end
	if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then
		return
	end
	local delta = input.Position - dragStart
	local maxX = math.max(0, gui.AbsoluteSize.X - panel.AbsoluteSize.X)
	local maxY = math.max(0, gui.AbsoluteSize.Y - panel.AbsoluteSize.Y)
	local x = math.clamp(dragOrigin.X + delta.X, 0, maxX)
	local y = math.clamp(dragOrigin.Y + delta.Y, 0, maxY)
	panel.Position = UDim2.fromOffset(x, y)
end)

local function setMinimized(value)
	minimized = value
	body.Visible = not minimized
	minimizeButton.Text = minimized and "+" or "-"
	TweenService:Create(panel, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		Size = UDim2.fromOffset(PANEL_WIDTH, minimized and HEADER_HEIGHT or PANEL_OPEN_HEIGHT),
	}):Play()
end

local function cleanupClones()
	for i = #clones, 1, -1 do
		if not clones[i].model.Parent then
			table.remove(clones, i)
		end
	end
end

local function getRoot()
	local character = player.Character
	return character and character:FindFirstChild("HumanoidRootPart"), character
end

local function animateIn(model)
	local fades = {}
	for _, descendant in ipairs(model:GetDescendants()) do
		if descendant:IsA("BasePart") or descendant:IsA("Decal") then
			table.insert(fades, { object = descendant, goal = descendant.Transparency })
			descendant.Transparency = 1
		end
	end

	local glow = Instance.new("Highlight")
	glow.FillColor = GLOW_COLOR
	glow.OutlineColor = GLOW_COLOR
	glow.FillTransparency = 0.35
	glow.OutlineTransparency = 0.1
	glow.Adornee = model
	glow.Parent = model

	local info = TweenInfo.new(FADE_IN_TIME, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
	for _, fade in ipairs(fades) do
		TweenService:Create(fade.object, info, { Transparency = fade.goal }):Play()
	end
	TweenService:Create(glow, TweenInfo.new(FADE_IN_TIME * 2), {
		FillTransparency = 1,
		OutlineTransparency = 1,
	}):Play()
	Debris:AddItem(glow, FADE_IN_TIME * 2 + 0.1)
end

local function animateOut(model)
	local info = TweenInfo.new(FADE_OUT_TIME, Enum.EasingStyle.Quad, Enum.EasingDirection.In)
	for _, descendant in ipairs(model:GetDescendants()) do
		if descendant:IsA("BasePart") or descendant:IsA("Decal") then
			TweenService:Create(descendant, info, { Transparency = 1 }):Play()
		end
	end
	Debris:AddItem(model, FADE_OUT_TIME + 0.1)
end

local function makeClone()
	cleanupClones()
	if #clones >= MAX_CLONES then
		updateStatus("Clone limit reached (" .. MAX_CLONES .. ").")
		return
	end

	local root, character = getRoot()
	if not root or not character then
		updateStatus("Your character is not loaded yet.")
		return
	end

	local wasArchivable = character.Archivable
	character.Archivable = true
	local model = character:Clone()
	character.Archivable = wasArchivable

	if not model then
		updateStatus("Could not copy your character.")
		return
	end

	for _, descendant in ipairs(model:GetDescendants()) do
		if (descendant:IsA("Script") or descendant:IsA("LocalScript")) and descendant.Name ~= "Animate" then
			descendant:Destroy()
		end
	end

	local index = #clones + 1
	local angle = math.rad(180 + (index - 1) * 40)
	local offset = Vector3.new(math.sin(angle), 0, math.cos(angle)) * FOLLOW_DISTANCE

	model.Name = "Clone" .. index
	model:PivotTo(root.CFrame * CFrame.new(offset))
	animateIn(model)
	model.Parent = workspace

	table.insert(clones, {
		model = model,
		offset = offset,
	})
	updateStatus()
end

local function clearClones()
	for _, entry in ipairs(clones) do
		if entry.model and entry.model.Parent then
			animateOut(entry.model)
		end
	end
	table.clear(clones)
	updateStatus("Clones cleared.")
end

local function setFollowing(value)
	following = value
	followButton.Text = following and "Follow Me: On" or "Follow Me: Off"
	followButton.BackgroundColor3 = following and COLOR_AMBER or COLOR_GREEN
end

minimizeButton.Activated:Connect(function()
	setMinimized(not minimized)
end)

makeCloneButton.Activated:Connect(makeClone)
clearButton.Activated:Connect(clearClones)
followButton.Activated:Connect(function()
	setFollowing(not following)
end)

RunService.Heartbeat:Connect(function(dt)
	if not following then
		return
	end

	repathTimer -= dt
	if repathTimer > 0 then
		return
	end
	repathTimer = REPATH_INTERVAL

	local root = getRoot()
	if not root then
		return
	end

	cleanupClones()
	for _, entry in ipairs(clones) do
		local model = entry.model
		local humanoid = model:FindFirstChildOfClass("Humanoid")
		local cloneRoot = model:FindFirstChild("HumanoidRootPart")
		if humanoid and cloneRoot and humanoid.Health > 0 then
			local target = (root.CFrame * CFrame.new(entry.offset)).Position
			if (cloneRoot.Position - target).Magnitude > FOLLOW_DISTANCE then
				humanoid:MoveTo(target)
			end
		end
	end
end)
