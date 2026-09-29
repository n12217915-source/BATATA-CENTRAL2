--=============================================================
--  ESP — re-aplica a cada 0.1s
--=============================================================
local Players           = game:GetService("Players")
local RunService        = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer       = Players.LocalPlayer
local Camera            = workspace.CurrentCamera

local Config = {
	Enabled    = false,
	ShowName   = true,
	ShowDist   = true,
	ShowHealth = true,
	BoxColor   = Color3.fromRGB(255,80,80),
	MaxDist    = 500,
}

local espFolder = Instance.new("Folder")
espFolder.Name = "BatataESP"
espFolder.Parent = game:GetService("CoreGui")

local active = {}  -- [player] = {highlight, billboard}

local function clearAll()
	for plr, data in pairs(active) do
		if data.highlight then data.highlight:Destroy() end
		if data.billboard then data.billboard:Destroy() end
	end
	active = {}
end

local function buildFor(plr)
	if plr == LocalPlayer then return end
	local char = plr.Character
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
	if not hrp then return end
	local hum = char:FindFirstChildOfClass("Humanoid")

	local hl = Instance.new("Highlight")
	hl.FillColor = Config.BoxColor
	hl.OutlineColor = Color3.new(1,1,1)
	hl.FillTransparency = 0.65
	hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
	hl.Adornee = char
	hl.Parent = espFolder

	local bb = Instance.new("BillboardGui")
	bb.Size = UDim2.fromOffset(200, 50)
	bb.StudsOffsetWorldSpace = Vector3.new(0, 3.2, 0)
	bb.AlwaysOnTop = true
	bb.Adornee = hrp
	bb.Parent = espFolder

	local label = Instance.new("TextLabel")
	label.BackgroundTransparency = 1
	label.Size = UDim2.new(1,0,1,0)
	label.Font = Enum.Font.GothamBold
	label.TextSize = 12
	label.TextColor3 = Config.BoxColor
	label.TextStrokeTransparency = 0
	label.Text = plr.Name
	label.Parent = bb

	active[plr] = { highlight = hl, billboard = bb, label = label, hum = hum, hrp = hrp }
end

local function updateLoop()
	-- remover antigos
	clearAll()
	if not Config.Enabled then return end
	for _, plr in ipairs(Players:GetPlayers()) do
		if plr ~= LocalPlayer then
			pcall(buildFor, plr)
		end
	end

	-- atualizar textos num loop curto
	local start = tick()
	while tick() - start < 0.1 and Config.Enabled do
		for plr, data in pairs(active) do
			if not data.label or not data.hrp or not data.hrp.Parent then continue end
			local parts = {}
			if Config.ShowName then parts[#parts+1] = plr.Name end
			if Config.ShowDist then
				local d = math.floor((Camera.CFrame.Position - data.hrp.Position).Magnitude)
				parts[#parts+1] = "["..d.."m]"
			end
			if Config.ShowHealth and data.hum then
				parts[#parts+1] = math.floor(data.hum.Health).."hp"
			end
			data.label.Text = table.concat(parts, " ")
			-- esconder se muito longe
			local dist = (Camera.CFrame.Position - data.hrp.Position).Magnitude
			data.label.Visible = dist <= Config.MaxDist
			data.highlight.Enabled = dist <= Config.MaxDist
		end
		task.wait(0.03)
	end
end

task.spawn(function()
	while task.wait(0.1) do
		if Config.Enabled then updateLoop() end
	end
end)

--=============================================================
--  Registro da aba (2.5s de delay)
--=============================================================
task.wait(2.5)
local api = ReplicatedStorage:WaitForChild("BatataHub_RegisterTab")

local ok, err = api:Invoke("Batata001", {
	Name = "ESP",
	BuildContent = function(page, ctx)
		local function makeToggle(text, y, initial, onChange)
			local btn = Instance.new("TextButton")
			btn.Position = UDim2.fromOffset(9,y)
			btn.Size = UDim2.new(1,-18,0,22)
			btn.BackgroundColor3 = ctx.colors.BUTTON or Color3.fromRGB(35,35,35)
			btn.Text = ""
			btn.BorderSizePixel = 0
			btn.Parent = page
			local c = Instance.new("UICorner") c.CornerRadius = UDim.new(0,5) c.Parent = btn
			local lbl = Instance.new("TextLabel")
			lbl.BackgroundTransparency = 1
			lbl.Position = UDim2.fromOffset(8,0)
			lbl.Size = UDim2.new(1,-50,1,0)
			lbl.Font = Enum.Font.Gotham
			lbl.TextSize = 11
			lbl.TextColor3 = ctx.colors.TEXT
			lbl.TextXAlignment = Enum.TextXAlignment.Left
			lbl.Text = text
			lbl.Parent = btn
			local st = Instance.new("TextLabel")
			st.BackgroundTransparency = 1
			st.Position = UDim2.new(1,-45,0,0)
			st.Size = UDim2.fromOffset(40,22)
			st.Font = Enum.Font.GothamBold
			st.TextSize = 11
			st.TextColor3 = initial and Color3.fromRGB(90,220,120) or Color3.fromRGB(220,90,90)
			st.Text = initial and "ON" or "OFF"
			st.Parent = btn
			btn.MouseButton1Click:Connect(function()
				initial = not initial
				st.Text = initial and "ON" or "OFF"
				st.TextColor3 = initial and Color3.fromRGB(90,220,120) or Color3.fromRGB(220,90,90)
				onChange(initial)
			end)
		end

		local t = Instance.new("TextLabel")
		t.BackgroundTransparency = 1
		t.Position = UDim2.fromOffset(9,8)
		t.Size = UDim2.new(1,-18,0,17)
		t.Font = Enum.Font.GothamBlack
		t.Text = "ESP"
		t.TextSize = 14
		t.TextColor3 = ctx.colors.TEXT
		t.TextXAlignment = Enum.TextXAlignment.Left
		t.Parent = page

		makeToggle("Ativado",     40, Config.Enabled,    function(v) Config.Enabled=v if not v then clearAll() end end)
		makeToggle("Nome",        68, Config.ShowName,   function(v) Config.ShowName=v end)
		makeToggle("Distância",   96, Config.ShowDist,   function(v) Config.ShowDist=v end)
		makeToggle("Vida",        124,Config.ShowHealth, function(v) Config.ShowHealth=v end)

		local info = Instance.new("TextLabel")
		info.BackgroundTransparency = 1
		info.Position = UDim2.fromOffset(9,160)
		info.Size = UDim2.new(1,-18,0,60)
		info.Font = Enum.Font.Gotham
		info.TextSize = 10
		info.TextColor3 = ctx.colors.SUBTEXT
		info.TextXAlignment = Enum.TextXAlignment.Left
		info.TextYAlignment = Enum.TextYAlignment.Top
		info.TextWrapped = true
		info.Text = "Re-aplica ESP em todos a cada 0.1s.\nSem dependência de team."
		info.Parent = page
	end
})

if not ok then warn("[ESP] registro falhou:", err) end
