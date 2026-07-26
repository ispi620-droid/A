local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")

local player = Players.LocalPlayer

--================================================
-- 1. 그림자
--================================================
local ok1, err1 = pcall(function()
    Lighting.Technology = Enum.Technology.Future
    Lighting.GlobalShadows = true
    Lighting.ShadowSoftness = 0.2
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("BasePart") then obj.CastShadow = true end
    end
end)
if not ok1 then warn("그림자 설정 에러:", err1) end

--================================================
-- 2. 크롬(빛 반사) - 맵 + 총(Tool)
--================================================
local ok2, err2 = pcall(function()
    Lighting.EnvironmentDiffuseScale = 1
    Lighting.EnvironmentSpecularScale = 1

    local materialReflectance = {
        [Enum.Material.Metal] = 0.7, [Enum.Material.DiamondPlate] = 0.7, [Enum.Material.Foil] = 0.7,
        [Enum.Material.Glass] = 0.6, [Enum.Material.Ice] = 0.6, [Enum.Material.Marble] = 0.5,
        [Enum.Material.SmoothPlastic] = 0.25, [Enum.Material.Plastic] = 0.15, [Enum.Material.Concrete] = 0.1,
    }

    local function applyMapChrome(obj)
        if obj:IsA("BasePart") then
            obj.Reflectance = materialReflectance[obj.Material] or 0.08
        end
    end
    for _, obj in ipairs(Workspace:GetDescendants()) do applyMapChrome(obj) end
    Workspace.DescendantAdded:Connect(applyMapChrome)

    local GUN_REFLECTANCE = 0.55
    local function applyGunChrome(tool)
        if not tool:IsA("Tool") and not tool:IsA("Accessory") then return end
        for _, part in ipairs(tool:GetDescendants()) do
            if part:IsA("BasePart") then part.Reflectance = GUN_REFLECTANCE end
        end
        tool.DescendantAdded:Connect(function(child)
            if child:IsA("BasePart") then child.Reflectance = GUN_REFLECTANCE end
        end)
    end
    local function hookContainer(container)
        for _, child in ipairs(container:GetChildren()) do applyGunChrome(child) end
        container.ChildAdded:Connect(applyGunChrome)
    end
    hookContainer(player:WaitForChild("Backpack"))
    if player.Character then hookContainer(player.Character) end
    player.CharacterAdded:Connect(hookContainer)
end)
if not ok2 then warn("크롬 반사 에러:", err2) end

--================================================
-- 3. PostEffect들 (Atmosphere 제외 - 안개 관련이라 뺌)
--================================================
local ok3, err3 = pcall(function()
    local cc = Lighting:FindFirstChild("RivalsStyleCC") or Instance.new("ColorCorrectionEffect")
    cc.Name = "RivalsStyleCC"
    cc.Contrast = 0.1
    cc.Saturation = 0.05
    cc.Parent = Lighting

    local bloom = Lighting:FindFirstChild("RivalsStyleBloom") or Instance.new("BloomEffect")
    bloom.Name = "RivalsStyleBloom"
    bloom.Intensity = 0.4
    bloom.Size = 24
    bloom.Threshold = 1.5
    bloom.Parent = Lighting

    local dof = Lighting:FindFirstChild("RivalsStyleDOF") or Instance.new("DepthOfFieldEffect")
    dof.Name = "RivalsStyleDOF"
    dof.FarIntensity = 0.2
    dof.FocusDistance = 50
    dof.InFocusRadius = 30
    dof.NearIntensity = 0
    dof.Parent = Lighting
end)
if not ok3 then warn("PostEffect 에러:", err3) end

--================================================
-- 4. 시간대별 조명 자동 전환 (안개 관련 값 제외)
--================================================
local colorCorrection = Lighting:FindFirstChild("RivalsStyleCC")
local DAY_LENGTH_SECONDS = 480 -- 8분 한 바퀴

local presets = {
    {time = 6,  ambient = Color3.fromRGB(75, 70, 65),  outdoorAmbient = Color3.fromRGB(100, 95, 88),  tint = Color3.fromRGB(255, 245, 235), brightness = 2},
    {time = 12, ambient = Color3.fromRGB(70, 70, 70),  outdoorAmbient = Color3.fromRGB(95, 95, 95),   tint = Color3.fromRGB(255, 255, 255), brightness = 2.2},
    {time = 18, ambient = Color3.fromRGB(68, 60, 55),  outdoorAmbient = Color3.fromRGB(90, 78, 68),   tint = Color3.fromRGB(255, 225, 200), brightness = 1.8},
    {time = 0,  ambient = Color3.fromRGB(85, 85, 100), outdoorAmbient = Color3.fromRGB(100, 100, 120), tint = Color3.fromRGB(210, 215, 230), brightness = 1.6},
}

local function lerpColor3(c1, c2, alpha)
    return Color3.new(c1.R + (c2.R - c1.R) * alpha, c1.G + (c2.G - c1.G) * alpha, c1.B + (c2.B - c1.B) * alpha)
end

local function getPresetBlend(clockTime)
    for i = 1, #presets do
        local current = presets[i]
        local next_ = presets[i + 1] or presets[1]
        local nextTime = next_.time
        if nextTime < current.time then nextTime = nextTime + 24 end
        local t = clockTime
        if t < current.time then t = t + 24 end
        if t >= current.time and t <= nextTime then
            local alpha = (t - current.time) / (nextTime - current.time)
            return current, next_, alpha
        end
    end
    return presets[1], presets[1], 0
end

Lighting.ClockTime = 12

RunService.Heartbeat:Connect(function(dt)
    local ok4, err4 = pcall(function()
        Lighting.ClockTime = (Lighting.ClockTime + (24 / DAY_LENGTH_SECONDS) * dt) % 24
        local current, next_, alpha = getPresetBlend(Lighting.ClockTime)

        Lighting.Ambient = lerpColor3(current.ambient, next_.ambient, alpha)
        Lighting.OutdoorAmbient = lerpColor3(current.outdoorAmbient, next_.outdoorAmbient, alpha)
        Lighting.Brightness = current.brightness + (next_.brightness - current.brightness) * alpha
        if colorCorrection then
            colorCorrection.TintColor = lerpColor3(current.tint, next_.tint, alpha)
        end
    end)
    if not ok4 then warn("시간전환 에러:", err4) end
end)

print("스크립트 로드 완료 (그림자/쉐이더/크롬/시간대전환, 안개 제외)")
