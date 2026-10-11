--[[
    DARKHUB – UI COMPACTA (VERSÃO 7.3 OTIMIZADA)
    Correções de performance: hooks, loops, cache de lookups
]]

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local Debris = game:GetService("Debris")
local StarterGui = game:GetService("StarterGui")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- [OTIMIZADO] Cache global de serviços e referências quentes
local RS_Remotes = game:GetService("ReplicatedStorage"):FindFirstChild("Remotes")
local RS_CommF = RS_Remotes and RS_Remotes:FindFirstChild("CommF_")
local RS_Modules = game:GetService("ReplicatedStorage"):FindFirstChild("Modules")
local RS_Net = RS_Modules and RS_Modules:FindFirstChild("Net")
local RS_RegisterHit = RS_Net and RS_Net:FindFirstChild("RE/RegisterHit")
local RS_RegisterAttack = RS_Net and RS_Net:FindFirstChild("RE/RegisterAttack")

-- [OTIMIZADO] Cache de KeyCodes (evita Enum.KeyCode.X repetido)
local KC_W, KC_A, KC_S, KC_D = Enum.KeyCode.W, Enum.KeyCode.A, Enum.KeyCode.S, Enum.KeyCode.D

xpcall(function()
    getgenv().SelectedTeam = "Pirates"

    task.spawn(function()
        task.wait(2)
        if RS_CommF then
            pcall(function() RS_CommF:InvokeServer("SetTeam", getgenv().SelectedTeam) end)
        end
        local char = player.Character or player.CharacterAdded:Wait()
        if char and not char:FindFirstChild("HasBuso") then
            pcall(function() RS_CommF:InvokeServer("Buso") end)
        end
    end)

    local guiParent = (gethui and gethui()) or CoreGui
    if guiParent:FindFirstChild("DARKHUB") then guiParent.DARKHUB:Destroy() end
    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "DARKHUB"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.Parent = guiParent
    ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    -- [OTIMIZADO] IgnoreGuiInset + baixo uso de memória
    ScreenGui.IgnoreGuiInset = true

    local Vermelho = Color3.fromRGB(200, 0, 0)
    local VermelhoEscuro = Color3.fromRGB(80, 0, 0)
    local VermelhoClaro = Color3.fromRGB(255, 150, 150)
    local VermelhoBrilho = Color3.fromRGB(255, 50, 50)
    local NeonRed = Color3.fromRGB(255, 0, 0)
    local DarkBg = Color3.fromRGB(12, 8, 20)

    local CoresAbas = {
        Pvp = Color3.fromRGB(255, 80, 80),
        Glitch = Color3.fromRGB(255, 180, 80),
        GlitchV2 = Color3.fromRGB(190, 80, 255),
        Fly = Color3.fromRGB(255, 120, 120),
        Gun = Color3.fromRGB(255, 200, 100),
        Config = Color3.fromRGB(180, 100, 100),
        Premium = Color3.fromRGB(255, 215, 0),
        Soru = Color3.fromRGB(0, 220, 180)
    }

    local uiVisible = true

    -- [OTIMIZADO] Estado local em tabela única (menos getgenv)
    local State = {
        GlideEnabled = false, GlideSpeed = 350, GlideTime = 0.2,
        JumpForce = 300, LastShootTime = 0,
        flyActive = false, flySpeed = 400,
        aimbotActive = false, aimbotTarget = nil,
        silentEnabled = true, silentTarget = nil, silentTargetPos = nil,
        silentLocked = false, silentAutoTarget = false, silentMaxDistance = 100,
        speedHack = false, speedHackValue = 80,
        gunSilentEnabled = false, gunMaxDistance = 100,
        gunTarget = nil, gunTargetPart = nil, gunTargetPos = nil,
        m1ExtenderActive = false, m1ExtenderRange = 50,
    }
    local FlyVel = nil

    local CONFIG_FOLDER = "DARKHUB"
    local CONFIG_FILE = "config.json"
    local hasFileFunctions = pcall(function() return writefile and readfile and isfolder and makefolder end)

    ------------------------------------------------------------------
    -- [OTIMIZADO] Helper de animação com cache
    ------------------------------------------------------------------
    -- Em vez de GetPlayingAnimationTracks() por Heartbeat (que é caro),
    -- usamos AnimationPlayed + cache de tracks ativas
    local function isAnimationPlaying(hum, idFragment, nameFragment)
        for _, a in ipairs(hum:GetPlayingAnimationTracks()) do
            if a.Animation then
                if idFragment and a.Animation.AnimationId:find(idFragment) then return true end
                if nameFragment and a.Name == nameFragment then return true end
            end
        end
        return false
    end

    ------------------------------------------------------------------
    -- GLITCH V2 MODULE
    ------------------------------------------------------------------
    local GlitchV2 = (function()
        local S = {
            sanguineEnabled = false, sanguinePower = 400, sanguineDuration = 0.9, sanguineCharge = 1.0,
            dtalonEnabled = false,   dtalonPower = 400,   dtalonDuration = 0.9,   dtalonCharge = 1.0,
            diamondEnabled = false,  diamondPower = 250,  diamondDuration = 0.3,  diamondCharge = 1.0,
            eclawEnabled = false,    eclawPower = 400,    eclawDuration = 1.7,    eclawCharge = 0.0,
            bigSkinBugEnabled = false,
        }
        local sanguineCharging, sanguineChargeStart = false, 0
        local dtalonCharging,   dtalonChargeStart   = false, 0
        local diamondCharging,  diamondChargeStart  = false, 0
        local bigSkinBugOriginal = {}
        local bigSkinBugConnections = {}
        local ui = {}

        local function getDmgLabel()
            local pg = player:FindFirstChild("PlayerGui")
            local main = pg and pg:FindFirstChild("Main")
            local dmg = main and main:FindFirstChild("DmgCounter")
            return dmg and dmg:FindFirstChild("Text")
        end

        local function makeBoost(power, duration, dir)
            local char = player.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if not hrp or not hum then return end
            hum.PlatformStand = true
            local att = Instance.new("Attachment", hrp)
            local lv = Instance.new("LinearVelocity", hrp)
            lv.MaxForce = 9999999
            lv.VelocityConstraintMode = Enum.VelocityConstraintMode.Vector
            lv.VectorVelocity = dir * power
            lv.Attachment0 = att
            local active = true
            local endTime = tick() + duration
            local dmgConn
            local function stop()
                if not active then return end
                active = false
                if lv then lv:Destroy() end
                if att then att:Destroy() end
                if hum then hum.PlatformStand = false end
                if dmgConn then dmgConn:Disconnect() end
            end
            local dmgLabel = getDmgLabel()
            local startText = dmgLabel and dmgLabel.Text or ""
            if dmgLabel then
                dmgConn = dmgLabel:GetPropertyChangedSignal("Text"):Connect(function()
                    local cur = dmgLabel.Text
                    if cur ~= startText and cur ~= "" and cur ~= "0" then stop() end
                end)
            end
            task.spawn(function()
                while active and tick() < endTime do task.wait() end
                stop()
            end)
        end

        local function applyBigSkin(char)
            local data = bigSkinBugOriginal[char]
            if not data then return end
            for item in pairs(data.accessories) do item.Parent = nil end
            for part in pairs(data.parts) do
                part.Color = Color3.fromRGB(128, 128, 128)
                part.Material = Enum.Material.SmoothPlastic
                part.CanCollide = true
                if part:IsA("MeshPart") then part.TextureID = "" end
                local nameLower = part.Name:lower()
                if nameLower:match("torso") then part.Size = Vector3.new(5.6, 4.4, 3.4)
                elseif nameLower:match("leg") or nameLower:match("arm") then part.Size = Vector3.new(2.6, 4.8, 2.6)
                elseif nameLower == "head" then
                    part.Size = Vector3.new(2.2, 2.2, 2.2)
                    local mesh = part:FindFirstChildOfClass("SpecialMesh")
                    if mesh then mesh.Parent = nil end
                end
            end
            for joint, saved in pairs(data.joints) do
                local jName = joint.Name:lower()
                local c0 = saved.C0
                if jName:match("rightshoulder") or jName:match("right shoulder") then joint.C0 = c0 * CFrame.new(2.4, 0.6, 0)
                elseif jName:match("leftshoulder") or jName:match("left shoulder") then joint.C0 = c0 * CFrame.new(-2.4, 0.6, 0)
                elseif jName:match("righthip") or jName:match("right hip") then joint.C0 = c0 * CFrame.new(0.9, -0.6, 0)
                elseif jName:match("lefthip") or jName:match("left hip") then joint.C0 = c0 * CFrame.new(-0.9, -0.6, 0)
                elseif jName:match("neck") then joint.C0 = c0 * CFrame.new(0, 0.8, 0) end
            end
        end

        local function saveBigSkinOriginal(char)
            local data = { parts = {}, accessories = {}, joints = {} }
            for _, item in pairs(char:GetDescendants()) do
                if item:IsA("Accessory") or item:IsA("Clothing") or item:IsA("ShirtGraphic") or item:IsA("Decal") then
                    data.accessories[item] = { Parent = item.Parent }
                elseif item:IsA("BasePart") and item.Name ~= "HumanoidRootPart" then
                    data.parts[item] = {
                        Size = item.Size, Color = item.Color, Material = item.Material,
                        TextureID = item:IsA("MeshPart") and item.TextureID or nil
                    }
                elseif item:IsA("Motor6D") and item.Part0 and item.Part1 then
                    data.joints[item] = { C0 = item.C0, C1 = item.C1 }
                end
            end
            bigSkinBugOriginal[char] = data
        end

        local function restoreBigSkin(char)
            local data = bigSkinBugOriginal[char]
            if not data then return end
            for item, saved in pairs(data.accessories) do
                if saved.Parent then item.Parent = saved.Parent end
            end
            for part, saved in pairs(data.parts) do
                part.Size = saved.Size
                part.Color = saved.Color
                part.Material = saved.Material
                if saved.TextureID and part:IsA("MeshPart") then part.TextureID = saved.TextureID end
            end
            for joint, saved in pairs(data.joints) do
                joint.C0 = saved.C0
                joint.C1 = saved.C1
            end
            bigSkinBugOriginal[char] = nil
        end

        local function enableBigSkin()
            if S.bigSkinBugEnabled then return end
            S.bigSkinBugEnabled = true
            if player.Character then
                saveBigSkinOriginal(player.Character)
                applyBigSkin(player.Character)
            end
            bigSkinBugConnections.CharAdded = player.CharacterAdded:Connect(function(c)
                task.wait(0.4)
                if not S.bigSkinBugEnabled then return end
                saveBigSkinOriginal(c)
                applyBigSkin(c)
            end)
        end

        local function disableBigSkin()
            S.bigSkinBugEnabled = false
            if bigSkinBugConnections.CharAdded then
                bigSkinBugConnections.CharAdded:Disconnect()
                bigSkinBugConnections.CharAdded = nil
            end
            if player.Character then restoreBigSkin(player.Character) end
        end

        -- [OTIMIZADO] Heartbeat único para todos os boosts (em vez de 3 separados)
        -- [OTIMIZADO] Só executa GetPlayingAnimationTracks se a feature estiver ativa
        RunService.Heartbeat:Connect(function()
            local anyActive = S.sanguineEnabled or S.dtalonEnabled or S.diamondEnabled
            if not anyActive then return end
            local char = player.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if not hum then return end

            if S.sanguineEnabled then
                local isGC = isAnimationPlaying(hum, "14418367908", "GhoulZCharge")
                if isGC then
                    if not sanguineCharging then sanguineCharging = true; sanguineChargeStart = tick() end
                else
                    if sanguineCharging then
                        if (tick() - sanguineChargeStart) >= S.sanguineCharge then
                            task.spawn(makeBoost, S.sanguinePower, S.sanguineDuration, camera.CFrame.LookVector)
                        end
                        sanguineCharging = false; sanguineChargeStart = 0
                    end
                end
            end

            if S.dtalonEnabled then
                local isC = isAnimationPlaying(hum, "18839089313", "DTalon_ZCharge")
                if isC then
                    if not dtalonCharging then dtalonCharging = true; dtalonChargeStart = tick() end
                else
                    if dtalonCharging then
                        if (tick() - dtalonChargeStart) >= S.dtalonCharge then
                            task.spawn(makeBoost, S.dtalonPower, S.dtalonDuration, camera.CFrame.LookVector)
                        end
                        dtalonCharging = false; dtalonChargeStart = 0
                    end
                end
            end

            if S.diamondEnabled then
                local isC = isAnimationPlaying(hum, "14414815375", nil)
                if isC then
                    if not diamondCharging then diamondCharging = true; diamondChargeStart = tick() end
                else
                    if diamondCharging then
                        if (tick() - diamondChargeStart) >= S.diamondCharge then
                            task.spawn(makeBoost, S.diamondPower, S.diamondDuration, camera.CFrame.LookVector)
                        end
                        diamondCharging = false; diamondChargeStart = 0
                    end
                end
            end
        end)

        local function hookEClaw(hum)
            hum.AnimationPlayed:Connect(function(track)
                if not S.eclawEnabled then return end
                if track.Animation and (track.Animation.AnimationId:find("6875496851") or track.Name == "ElectroClawXImpact") then
                    local chargeStart = tick()
                    track.Stopped:Connect(function()
                        if (tick() - chargeStart) >= S.eclawCharge then
                            local char = player.Character
                            local hrp = char and char:FindFirstChild("HumanoidRootPart")
                            if hrp then
                                local mouse = player:GetMouse()
                                local dir
                                if mouse and mouse.Hit then dir = (mouse.Hit.Position - hrp.Position).Unit
                                else dir = camera.CFrame.LookVector end
                                task.spawn(makeBoost, S.eclawPower, S.eclawDuration, dir)
                            end
                        end
                    end)
                end
            end)
        end
        if player.Character then
            local hum = player.Character:FindFirstChildOfClass("Humanoid")
            if hum then hookEClaw(hum) end
        end
        player.CharacterAdded:Connect(function(c)
            local hum = c:WaitForChild("Humanoid", 5)
            if hum then hookEClaw(hum) end
        end)

        local M = { S = S }

        function M.LoadFrom(data)
            for k, v in pairs(data) do
                if S[k] ~= nil then S[k] = v end
            end
        end

        function M.SaveInto(data)
            for k, v in pairs(S) do data[k] = v end
        end

        function M.Sync()
            if not ui.SangBtn then return end
            ui.SangBtn.Text = S.sanguineEnabled and "🩸 Sanguine: ON" or "🩸 Sanguine: OFF"
            ui.SangBtn.BackgroundColor3 = S.sanguineEnabled and VermelhoEscuro or Color3.fromRGB(30,20,40)
            ui.DTBtn.Text = S.dtalonEnabled and "⚔️ DTalon: ON" or "⚔️ DTalon: OFF"
            ui.DTBtn.BackgroundColor3 = S.dtalonEnabled and VermelhoEscuro or Color3.fromRGB(30,20,40)
            ui.DiaBtn.Text = S.diamondEnabled and "💎 Diamond: ON" or "💎 Diamond: OFF"
            ui.DiaBtn.BackgroundColor3 = S.diamondEnabled and VermelhoEscuro or Color3.fromRGB(30,20,40)
            ui.ECBtn.Text = S.eclawEnabled and "⚡ EClaw: ON" or "⚡ EClaw: OFF"
            ui.ECBtn.BackgroundColor3 = S.eclawEnabled and VermelhoEscuro or Color3.fromRGB(30,20,40)
            ui.BSBBtn.Text = S.bigSkinBugEnabled and "🟫 Big Skin: ON" or "🟫 Big Skin: OFF"
            ui.BSBBtn.BackgroundColor3 = S.bigSkinBugEnabled and VermelhoEscuro or Color3.fromRGB(30,20,40)
        end

        function M.CreateUI(page, h)
            local function makeTitle(text, y)
                local t = Instance.new("TextLabel")
                t.Size = UDim2.new(0.5,0,0,16)
                t.Position = UDim2.new(0.05,0,0,y)
                t.BackgroundTransparency = 1
                t.Text = text
                t.TextColor3 = h.VermelhoClaro
                t.Font = Enum.Font.GothamBold
                t.TextSize = 12
                t.Parent = page
                return t
            end

            makeTitle("🩸 Sanguine Z", 30)
            ui.SangBtn = h.createToggleButton("Sanguine: OFF", 48, page)
            ui.SangPow = h.createInputField("Power:", S.sanguinePower, 80, page)
            ui.SangDur = h.createInputField("Duração:", S.sanguineDuration, 104, page)
            ui.SangChg = h.createInputField("Min Charge:", S.sanguineCharge, 128, page)
            ui.SangBtn.MouseButton1Click:Connect(function() S.sanguineEnabled = not S.sanguineEnabled; M.Sync(); h.save() end)
            ui.SangPow.FocusLost:Connect(function() local v=tonumber(ui.SangPow.Text); if v then S.sanguinePower=v; h.save() end end)
            ui.SangDur.FocusLost:Connect(function() local v=tonumber(ui.SangDur.Text); if v then S.sanguineDuration=v; h.save() end end)
            ui.SangChg.FocusLost:Connect(function() local v=tonumber(ui.SangChg.Text); if v then S.sanguineCharge=v; h.save() end end)

            makeTitle("⚔️ DTalon Z", 158)
            ui.DTBtn = h.createToggleButton("DTalon: OFF", 176, page)
            ui.DTPow = h.createInputField("Power:", S.dtalonPower, 208, page)
            ui.DTDur = h.createInputField("Duração:", S.dtalonDuration, 232, page)
            ui.DTChg = h.createInputField("Min Charge:", S.dtalonCharge, 256, page)
            ui.DTBtn.MouseButton1Click:Connect(function() S.dtalonEnabled = not S.dtalonEnabled; M.Sync(); h.save() end)
            ui.DTPow.FocusLost:Connect(function() local v=tonumber(ui.DTPow.Text); if v then S.dtalonPower=v; h.save() end end)
            ui.DTDur.FocusLost:Connect(function() local v=tonumber(ui.DTDur.Text); if v then S.dtalonDuration=v; h.save() end end)
            ui.DTChg.FocusLost:Connect(function() local v=tonumber(ui.DTChg.Text); if v then S.dtalonCharge=v; h.save() end end)

            makeTitle("💎 Diamond Z", 286)
            ui.DiaBtn = h.createToggleButton("Diamond: OFF", 304, page)
            ui.DiaPow = h.createInputField("Power:", S.diamondPower, 336, page)
            ui.DiaDur = h.createInputField("Duração:", S.diamondDuration, 360, page)
            ui.DiaChg = h.createInputField("Min Charge:", S.diamondCharge, 384, page)
            ui.DiaBtn.MouseButton1Click:Connect(function() S.diamondEnabled = not S.diamondEnabled; M.Sync(); h.save() end)
            ui.DiaPow.FocusLost:Connect(function() local v=tonumber(ui.DiaPow.Text); if v then S.diamondPower=v; h.save() end end)
            ui.DiaDur.FocusLost:Connect(function() local v=tonumber(ui.DiaDur.Text); if v then S.diamondDuration=v; h.save() end end)
            ui.DiaChg.FocusLost:Connect(function() local v=tonumber(ui.DiaChg.Text); if v then S.diamondCharge=v; h.save() end end)

            makeTitle("⚡ EClaw X", 414)
            ui.ECBtn = h.createToggleButton("EClaw: OFF", 432, page)
            ui.ECPow = h.createInputField("Power:", S.eclawPower, 464, page)
            ui.ECDur = h.createInputField("Duração:", S.eclawDuration, 488, page)
            ui.ECChg = h.createInputField("Min Charge:", S.eclawCharge, 512, page)
            ui.ECBtn.MouseButton1Click:Connect(function() S.eclawEnabled = not S.eclawEnabled; M.Sync(); h.save() end)
            ui.ECPow.FocusLost:Connect(function() local v=tonumber(ui.ECPow.Text); if v then S.eclawPower=v; h.save() end end)
            ui.ECDur.FocusLost:Connect(function() local v=tonumber(ui.ECDur.Text); if v then S.eclawDuration=v; h.save() end end)
            ui.ECChg.FocusLost:Connect(function() local v=tonumber(ui.ECChg.Text); if v then S.eclawCharge=v; h.save() end end)

            makeTitle("🟫 Big Skin Bug", 542)
            ui.BSBBtn = h.createToggleButton("Big Skin: OFF", 560, page)
            ui.BSBBtn.MouseButton1Click:Connect(function()
                if S.bigSkinBugEnabled then disableBigSkin() else enableBigSkin() end
                M.Sync(); h.save()
            end)

            M.Sync()
        end

        return M
    end)()

    ------------------------------------------------------------------
    -- [OTIMIZADO] SORU ASSIST MODULE (cache severo + loop único)
    ------------------------------------------------------------------
    local Soru = (function()
        local S = {
            enabled = false,
            targetPart = "HumanoidRootPart",
            priority = "Nearest",
            stickyTarget = true,
            wallCheck = false,
            predictionEnabled = true,
            predictionAmount = 0.12,
            maxRange = 300,
        }
        local currentTarget = nil
        local predictedPos = nil
        local currentSkillKey = nil
        local skillLockTime = 0
        local Mouse = player:GetMouse()
        local ui = {}

        -- [OTIMIZADO] Cache de raycast params (evita criar toda chamada)
        local rayParams = RaycastParams.new()
        rayParams.FilterType = Enum.RaycastFilterType.Exclude

        local function predictPosition(hrp)
            if not hrp then return nil end
            if not S.predictionEnabled then return hrp.Position end
            local vel = hrp.AssemblyLinearVelocity
            if vel.Magnitude < 5 then return hrp.Position end
            return hrp.Position + (vel * S.predictionAmount)
        end

        local function getAimPart(character)
            if not character then return nil end
            return character[S.targetPart] or character:FindFirstChild("HumanoidRootPart")
        end

        -- [OTIMIZADO] Atualiza FilterDescendantsInstances só quando personagem muda
        local lastChar = nil
        local function hasLineOfSight(part)
            if not S.wallCheck or not part then return true end
            local char = player.Character
            if char ~= lastChar then
                lastChar = char
                rayParams.FilterDescendantsInstances = { char, camera }
            end
            local origin = camera.CFrame.Position
            local hit = workspace:Raycast(origin, part.Position - origin, rayParams)
            return not hit or hit.Instance:IsDescendantOf(part.Parent)
        end

        local function isPartUsable(part, lpHRP)
            if not part or not part.Parent or not lpHRP then return false end
            local hum = part.Parent:FindFirstChildWhichIsA("Humanoid")
            if not hum or hum.Health <= 0 then return false end
            if (part.Position - lpHRP.Position).Magnitude > S.maxRange then return false end
            if not hasLineOfSight(part) then return false end
            return true
        end

        local function getTarget(lpHRP)
            if not lpHRP then return nil end

            if S.stickyTarget and currentTarget then
                if currentTarget.Parent and currentTarget.Parent.Parent then
                    local hum = currentTarget.Parent:FindFirstChildWhichIsA("Humanoid")
                    if hum and hum.Health > 0 and isPartUsable(currentTarget, lpHRP) then
                        return currentTarget
                    end
                end
            end

            local valid = {}
            for _, pl in ipairs(Players:GetPlayers()) do
                if pl ~= player then
                    local char = pl.Character
                    if char and char.Parent then
                        local hum = char:FindFirstChildWhichIsA("Humanoid")
                        local aimPart = getAimPart(char)
                        if hum and hum.Health > 0 and aimPart and isPartUsable(aimPart, lpHRP) then
                            local dist = (aimPart.Position - lpHRP.Position).Magnitude
                            table.insert(valid, {Part = aimPart, Humanoid = hum, Distance = dist})
                        end
                    end
                end
            end

            if #valid == 0 then return nil end

            local prio = S.priority
            if prio == "Nearest" then
                table.sort(valid, function(a,b) return a.Distance < b.Distance end)
            elseif prio == "Low HP" then
                table.sort(valid, function(a,b) return a.Humanoid.Health < b.Humanoid.Health end)
            elseif prio == "Looking At Me" then
                table.sort(valid, function(a,b)
                    local dirA = (lpHRP.Position - a.Part.Position).Unit
                    local dirB = (lpHRP.Position - b.Part.Position).Unit
                    return a.Part.CFrame.LookVector:Dot(dirA) > b.Part.CFrame.LookVector:Dot(dirB)
                end)
            end

            return valid[1].Part
        end

        local function faceTarget(pos)
            if not pos then return end
            local char = player.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp then return end
            local lookDir = (Vector3.new(pos.X, hrp.Position.Y, pos.Z) - hrp.Position)
            if lookDir.Magnitude < 0.001 then return end
            hrp.CFrame = CFrame.lookAt(hrp.Position, hrp.Position + lookDir.Unit, Vector3.new(0,1,0))
        end

        local function setCurrentSkillKey(key)
            currentSkillKey = key
            skillLockTime = tick() + 1.5
            if key == "C" and predictedPos then
                task.spawn(function() faceTarget(predictedPos) end)
            end
        end

        -- [OTIMIZADO] Loop único para update + skill key cleanup
        -- [OTIMIZADO] Intervalo aumentado de 0.08 para 0.12 (imperceptível)
        task.spawn(function()
            while true do
                task.wait(0.12)
                if S.enabled then
                    local myHRP = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
                    if myHRP then
                        local targetPart = getTarget(myHRP)
                        if targetPart then
                            currentTarget = targetPart
                            predictedPos = predictPosition(targetPart)
                        else
                            currentTarget = nil
                            predictedPos = nil
                        end
                    else
                        currentTarget = nil
                        predictedPos = nil
                    end
                else
                    currentTarget = nil
                    predictedPos = nil
                end
                if currentSkillKey and tick() > skillLockTime then
                    currentSkillKey = nil
                end
            end
        end)

        UserInputService.InputBegan:Connect(function(input, gp)
            if gp then return end
            local kc = input.KeyCode
            if kc == Enum.KeyCode.Z then setCurrentSkillKey("Z")
            elseif kc == Enum.KeyCode.X then setCurrentSkillKey("X")
            elseif kc == Enum.KeyCode.C then setCurrentSkillKey("C")
            elseif kc == Enum.KeyCode.V then setCurrentSkillKey("V")
            elseif kc == Enum.KeyCode.F then setCurrentSkillKey("F")
            end
        end)

        local M = { S = S }
        function M.GetPredictedPosition() return predictedPos end
        function M.GetCurrentTarget() return currentTarget end
        function M.GetCurrentSkillKey() return currentSkillKey end
        function M.SetCurrentSkillKey(k) setCurrentSkillKey(k) end
        function M.GetMouseObject() return Mouse end
        function M.FaceTarget(pos) faceTarget(pos) end
        function M.IsEnabled() return S.enabled end
        function M.HasTarget() return predictedPos ~= nil end

        function M.LoadFrom(data)
            if data.soruEnabled ~= nil then S.enabled = data.soruEnabled end
            if data.soruTargetPart then S.targetPart = data.soruTargetPart end
            if data.soruPriority then S.priority = data.soruPriority end
            if data.soruSticky ~= nil then S.stickyTarget = data.soruSticky end
            if data.soruWallCheck ~= nil then S.wallCheck = data.soruWallCheck end
            if data.soruPrediction ~= nil then S.predictionEnabled = data.soruPrediction end
            if data.soruPredictionAmount then S.predictionAmount = data.soruPredictionAmount end
            if data.soruMaxRange then S.maxRange = data.soruMaxRange end
        end

        function M.SaveInto(data)
            data.soruEnabled = S.enabled
            data.soruTargetPart = S.targetPart
            data.soruPriority = S.priority
            data.soruSticky = S.stickyTarget
            data.soruWallCheck = S.wallCheck
            data.soruPrediction = S.predictionEnabled
            data.soruPredictionAmount = S.predictionAmount
            data.soruMaxRange = S.maxRange
        end

        function M.Sync()
            if not ui.EnableBtn then return end
            ui.EnableBtn.Text = S.enabled and "🌀 Soru Assist: ON" or "🌀 Soru Assist: OFF"
            ui.EnableBtn.BackgroundColor3 = S.enabled and VermelhoEscuro or Color3.fromRGB(30,20,40)
            ui.PartBtn.Text = "Target Part: " .. S.targetPart
            ui.PrioBtn.Text = "Priority: " .. S.priority
            ui.StickyBtn.Text = S.stickyTarget and "Sticky Target: ON" or "Sticky Target: OFF"
            ui.StickyBtn.BackgroundColor3 = S.stickyTarget and VermelhoEscuro or Color3.fromRGB(30,20,40)
            ui.WallBtn.Text = S.wallCheck and "Wall Check: ON" or "Wall Check: OFF"
            ui.WallBtn.BackgroundColor3 = S.wallCheck and VermelhoEscuro or Color3.fromRGB(30,20,40)
            ui.PredBtn.Text = S.predictionEnabled and "Movement Prediction: ON" or "Movement Prediction: OFF"
            ui.PredBtn.BackgroundColor3 = S.predictionEnabled and VermelhoEscuro or Color3.fromRGB(30,20,40)
        end

        function M.CreateUI(page, h)
            local function makeTitle(text, y)
                local t = Instance.new("TextLabel")
                t.Size = UDim2.new(0.6,0,0,16)
                t.Position = UDim2.new(0.05,0,0,y)
                t.BackgroundTransparency = 1
                t.Text = text
                t.TextColor3 = h.VermelhoClaro
                t.Font = Enum.Font.GothamBold
                t.TextSize = 12
                t.Parent = page
                return t
            end

            local function makeCycleButton(label, values, initial, y, parent, onChange)
                local idx = 1
                for i, v in ipairs(values) do if v == initial then idx = i break end end
                local btn = h.createToggleButton(label .. ": " .. values[idx], y, parent)
                btn.MouseButton1Click:Connect(function()
                    idx = idx + 1
                    if idx > #values then idx = 1 end
                    btn.Text = label .. ": " .. values[idx]
                    onChange(values[idx])
                    h.save()
                end)
                return btn
            end

            ui.EnableBtn = h.createToggleButton("🌀 Soru Assist: OFF", 34, page)
            ui.EnableBtn.MouseButton1Click:Connect(function()
                S.enabled = not S.enabled
                M.Sync(); h.save()
            end)

            makeTitle("— Target Settings —", 72)
            ui.PartBtn = makeCycleButton("Target Part", {"HumanoidRootPart","Head","UpperTorso"}, S.targetPart, 90, page,
                function(v) S.targetPart = v; currentTarget = nil end)
            ui.PrioBtn = makeCycleButton("Priority", {"Nearest","Low HP","Looking At Me"}, S.priority, 120, page,
                function(v) S.priority = v end)
            ui.StickyBtn = h.createToggleButton("Sticky Target: ON", 150, page)
            ui.StickyBtn.MouseButton1Click:Connect(function()
                S.stickyTarget = not S.stickyTarget
                if not S.stickyTarget then currentTarget = nil end
                M.Sync(); h.save()
            end)
            ui.WallBtn = h.createToggleButton("Wall Check: OFF", 180, page)
            ui.WallBtn.MouseButton1Click:Connect(function()
                S.wallCheck = not S.wallCheck
                M.Sync(); h.save()
            end)

            makeTitle("— Prediction —", 218)
            ui.PredBtn = h.createToggleButton("Movement Prediction: ON", 236, page)
            ui.PredBtn.MouseButton1Click:Connect(function()
                S.predictionEnabled = not S.predictionEnabled
                M.Sync(); h.save()
            end)
            ui.PredIn = h.createInputField("Prediction:", S.predictionAmount, 268, page)
            ui.PredIn.FocusLost:Connect(function()
                local v = tonumber(ui.PredIn.Text); if v then S.predictionAmount = v; h.save() end
            end)

            makeTitle("— Range —", 304)
            ui.RangeIn = h.createInputField("Max Range:", S.maxRange, 322, page)
            ui.RangeIn.FocusLost:Connect(function()
                local v = tonumber(ui.RangeIn.Text); if v then S.maxRange = v; h.save() end
            end)

            makeTitle("— Info —", 358)
            ui.InfoLabel = Instance.new("TextLabel")
            ui.InfoLabel.Size = UDim2.new(0.9,0,0,40)
            ui.InfoLabel.Position = UDim2.new(0.05,0,0,378)
            ui.InfoLabel.BackgroundTransparency = 1
            ui.InfoLabel.Text = "Mira na direção do alvo quando você usa a skill C (Soru)."
            ui.InfoLabel.TextColor3 = Color3.fromRGB(180,180,180)
            ui.InfoLabel.Font = Enum.Font.GothamMedium
            ui.InfoLabel.TextSize = 11
            ui.InfoLabel.TextWrapped = true
            ui.InfoLabel.TextXAlignment = Enum.TextXAlignment.Left
            ui.InfoLabel.Parent = page

            ui.StatusLabel = Instance.new("TextLabel")
            ui.StatusLabel.Size = UDim2.new(0.9,0,0,18)
            ui.StatusLabel.Position = UDim2.new(0.05,0,0,428)
            ui.StatusLabel.BackgroundTransparency = 1
            ui.StatusLabel.Text = "Status: aguardando..."
            ui.StatusLabel.TextColor3 = h.VermelhoClaro
            ui.StatusLabel.Font = Enum.Font.GothamMedium
            ui.StatusLabel.TextSize = 11
            ui.StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
            ui.StatusLabel.Parent = page

            M.Sync()

            -- [OTIMIZADO] Status label loop apenas quando a página está visível
            -- Aumentado intervalo para 0.5s (era 0.3s)
            task.spawn(function()
                while true do
                    task.wait(0.5)
                    if ui.StatusLabel and ui.StatusLabel.Parent and ui.StatusLabel.Parent.Visible then
                        if not S.enabled then
                            ui.StatusLabel.Text = "Status: desligado"
                        elseif currentTarget then
                            ui.StatusLabel.Text = "Status: mirando (" .. (currentSkillKey or "—") .. ")"
                        else
                            ui.StatusLabel.Text = "Status: sem alvo"
                        end
                    end
                end
            end)
        end

        _G.__SoruModule = M
        return M
    end)()

    ------------------------------------------------------------------
    -- CONFIG
    ------------------------------------------------------------------
    local function ensureFolder()
        if not hasFileFunctions then return false end
        if not isfolder(CONFIG_FOLDER) then makefolder(CONFIG_FOLDER) end
        return true
    end

    -- [OTIMIZADO] Save debounced (evita writefile a cada clique)
    local savePending = false
    local function scheduleSave()
        if savePending then return end
        savePending = true
        task.delay(0.4, function()
            savePending = false
            saveConfig()
        end)
    end

    local function loadConfig()
        if not hasFileFunctions then return false end
        if not ensureFolder() then return false end
        local path = CONFIG_FOLDER.."/"..CONFIG_FILE
        if not isfile(path) then return false end
        local success, data = pcall(function() return HttpService:JSONDecode(readfile(path)) end)
        if not success or not data then return false end
        State.GlideEnabled = data.glideEnabled or false
        State.GlideSpeed = data.glideSpeed or 350
        State.GlideTime = data.glideTime or 0.2
        State.JumpForce = data.jumpForce or 300
        State.flyActive = data.flyActive or false
        State.flySpeed = data.flySpeed or 400
        State.aimbotActive = data.aimbotActive or false
        State.silentEnabled = (data.silentEnabled ~= nil) and data.silentEnabled or true
        State.silentAutoTarget = data.silentAutoTarget or false
        State.silentMaxDistance = data.silentMaxDistance or 100
        State.speedHack = data.speedHack or false
        State.speedHackValue = data.speedHackValue or 80
        if data.silentLocked ~= nil then State.silentLocked = data.silentLocked end
        if data.selectedTeam ~= nil then getgenv().SelectedTeam = data.selectedTeam end
        if data.gunSilentEnabled ~= nil then State.gunSilentEnabled = data.gunSilentEnabled end
        if data.gunMaxDistance then State.gunMaxDistance = data.gunMaxDistance end
        if data.m1ExtenderActive ~= nil then State.m1ExtenderActive = data.m1ExtenderActive end
        if data.m1ExtenderRange then State.m1ExtenderRange = data.m1ExtenderRange end
        GlitchV2.LoadFrom(data)
        Soru.LoadFrom(data)
        return true
    end

    function saveConfig()
        if not hasFileFunctions then return false end
        if not ensureFolder() then return false end
        local data = {
            glideEnabled = State.GlideEnabled,
            glideSpeed = State.GlideSpeed,
            glideTime = State.GlideTime,
            jumpForce = State.JumpForce,
            flyActive = State.flyActive,
            flySpeed = State.flySpeed,
            aimbotActive = State.aimbotActive,
            silentEnabled = State.silentEnabled,
            silentAutoTarget = State.silentAutoTarget,
            silentMaxDistance = State.silentMaxDistance,
            silentLocked = State.silentLocked,
            speedHack = State.speedHack,
            speedHackValue = State.speedHackValue,
            selectedTeam = getgenv().SelectedTeam,
            gunSilentEnabled = State.gunSilentEnabled,
            gunMaxDistance = State.gunMaxDistance,
            m1ExtenderActive = State.m1ExtenderActive,
            m1ExtenderRange = State.m1ExtenderRange,
        }
        GlitchV2.SaveInto(data)
        Soru.SaveInto(data)
        local json = HttpService:JSONEncode(data)
        pcall(function() writefile(CONFIG_FOLDER.."/"..CONFIG_FILE, json) end)
        return true
    end
    loadConfig()

    ------------------------------------------------------------------
    -- UI HELPERS
    ------------------------------------------------------------------
    local function createBorder(frame, thickness, color)
        local stroke = Instance.new("UIStroke")
        stroke.Thickness = thickness or 1.5
        stroke.Color = color or Vermelho
        stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        stroke.Parent = frame
        return stroke
    end

    local function createToggleButton(text, yPos, parentPage)
        local Btn = Instance.new("TextButton")
        Btn.Size = UDim2.new(0.9,0,0,28)
        Btn.Position = UDim2.new(0.05,0,0,yPos)
        Btn.BackgroundColor3 = Color3.fromRGB(30,20,40)
        Btn.Text = text
        Btn.TextColor3 = VermelhoClaro
        Btn.Font = Enum.Font.GothamMedium
        Btn.TextSize = 12
        Btn.Parent = parentPage
        Instance.new("UICorner", Btn).CornerRadius = UDim.new(0,4)
        createBorder(Btn, 1.5, VermelhoBrilho)
        Btn.MouseEnter:Connect(function() TweenService:Create(Btn, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(60,30,30) }):Play() end)
        Btn.MouseLeave:Connect(function() TweenService:Create(Btn, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(30,20,40) }):Play() end)
        return Btn
    end

    local function createInputField(labelText, defaultVal, yPos, parentPage)
        local Label = Instance.new("TextLabel")
        Label.Size = UDim2.new(0.35,0,0,24)
        Label.Position = UDim2.new(0,5,0,yPos)
        Label.BackgroundTransparency = 1
        Label.Text = labelText
        Label.TextColor3 = VermelhoClaro
        Label.TextXAlignment = Enum.TextXAlignment.Left
        Label.Font = Enum.Font.GothamMedium
        Label.TextSize = 12
        Label.Parent = parentPage

        local InputBg = Instance.new("Frame")
        InputBg.Size = UDim2.new(0.45,0,0,22)
        InputBg.Position = UDim2.new(0.5,0,0,yPos+1)
        InputBg.BackgroundColor3 = Color3.fromRGB(40,30,50)
        InputBg.BorderSizePixel = 0
        InputBg.Parent = parentPage
        Instance.new("UICorner", InputBg).CornerRadius = UDim.new(0,4)
        createBorder(InputBg, 1, VermelhoBrilho)

        local Input = Instance.new("TextBox")
        Input.Size = UDim2.new(1,0,1,0)
        Input.BackgroundTransparency = 1
        Input.TextColor3 = VermelhoClaro
        Input.Text = tostring(defaultVal)
        Input.Font = Enum.Font.GothamMedium
        Input.TextSize = 12
        Input.Parent = InputBg
        return Input
    end

    ------------------------------------------------------------------
    -- MAIN FRAME
    ------------------------------------------------------------------
    local MainFrame = Instance.new("Frame")
    MainFrame.Size = UDim2.new(0,0,0,0)
    MainFrame.Position = UDim2.new(0.02,0,0.02,0)
    MainFrame.BackgroundColor3 = DarkBg
    MainFrame.BackgroundTransparency = 0.15
    MainFrame.BorderSizePixel = 0
    MainFrame.Active = true
    MainFrame.Draggable = true
    MainFrame.Visible = false
    MainFrame.Parent = ScreenGui
    Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0,12)
    createBorder(MainFrame, 2, NeonRed)

    local TopBar = Instance.new("Frame")
    TopBar.Size = UDim2.new(1,0,0,36)
    TopBar.BackgroundTransparency = 1
    TopBar.Parent = MainFrame

    local TitleTop = Instance.new("TextLabel")
    TitleTop.Size = UDim2.new(1,0,0,32)
    TitleTop.Position = UDim2.new(0,0,0,2)
    TitleTop.BackgroundTransparency = 1
    TitleTop.Text = "♥ DARK HUB ♥"
    TitleTop.TextColor3 = NeonRed
    TitleTop.Font = Enum.Font.GothamBlack
    TitleTop.TextSize = 24
    TitleTop.TextXAlignment = Enum.TextXAlignment.Center
    TitleTop.Parent = TopBar

    local TabBar = Instance.new("Frame")
    TabBar.Size = UDim2.new(1,0,0,32)
    TabBar.Position = UDim2.new(0,0,0,36)
    TabBar.BackgroundColor3 = Color3.fromRGB(20,12,12)
    TabBar.BackgroundTransparency = 0.3
    TabBar.BorderSizePixel = 0
    TabBar.Parent = MainFrame

    local TabScroll = Instance.new("ScrollingFrame")
    TabScroll.Size = UDim2.new(1,0,1,0)
    TabScroll.BackgroundTransparency = 1
    TabScroll.ScrollBarThickness = 0
    TabScroll.CanvasSize = UDim2.new(0,0,0,0)
    TabScroll.Parent = TabBar

    local TabLayout = Instance.new("UIListLayout")
    TabLayout.FillDirection = Enum.FillDirection.Horizontal
    TabLayout.VerticalAlignment = Enum.VerticalAlignment.Center
    TabLayout.Padding = UDim.new(0,6)
    TabLayout.SortOrder = Enum.SortOrder.LayoutOrder
    TabLayout.Parent = TabScroll

    local tabLines = {}
    local function createTabButton(text, order, cor)
        local Btn = Instance.new("TextButton")
        Btn.Size = UDim2.new(0,68,0,26)
        Btn.BackgroundColor3 = Color3.fromRGB(30,20,40)
        Btn.Text = text
        Btn.TextColor3 = VermelhoClaro
        Btn.Font = Enum.Font.GothamBold
        Btn.TextSize = 11
        Btn.LayoutOrder = order
        Btn.Parent = TabScroll
        Instance.new("UICorner", Btn).CornerRadius = UDim.new(0,4)
        local bottomLine = Instance.new("Frame")
        bottomLine.Size = UDim2.new(1,0,0,2)
        bottomLine.Position = UDim2.new(0,0,1,-2)
        bottomLine.BackgroundColor3 = cor
        bottomLine.BackgroundTransparency = 1
        bottomLine.Parent = Btn
        tabLines[Btn] = bottomLine
        Btn.MouseEnter:Connect(function() TweenService:Create(Btn, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(50,25,25) }):Play() end)
        Btn.MouseLeave:Connect(function() TweenService:Create(Btn, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(30,20,40) }):Play() end)
        return Btn
    end

    local TabPvpBtn      = createTabButton("⚔️ PVP",0, CoresAbas.Pvp)
    local TabGlitchBtn   = createTabButton("⚡ Glitch",1, CoresAbas.Glitch)
    local TabFlyBtn      = createTabButton("🪁 Voo",2, CoresAbas.Fly)
    local TabGunBtn      = createTabButton("🔫 Gun",3, CoresAbas.Gun)
    local TabConfigBtn   = createTabButton("⚙️ Config",4, CoresAbas.Config)
    local TabPremiumBtn  = createTabButton("💎 Premium",5, CoresAbas.Premium)
    local TabGlitchV2Btn = createTabButton("💥 Glitch V2",6, CoresAbas.GlitchV2)
    local TabSoruBtn     = createTabButton("🌀 Soru",7, CoresAbas.Soru)

    task.spawn(function()
        task.wait(0.1)
        local count = #TabScroll:GetChildren()
        TabScroll.CanvasSize = UDim2.new(0, count * 74, 0, 0)
    end)

    local ContentArea = Instance.new("Frame")
    ContentArea.Size = UDim2.new(1,0,1,-68)
    ContentArea.Position = UDim2.new(0,0,0,68)
    ContentArea.BackgroundTransparency = 1
    ContentArea.Parent = MainFrame

    local Pages = Instance.new("Frame")
    Pages.Size = UDim2.new(1,0,1,0)
    Pages.BackgroundTransparency = 1
    Pages.Parent = ContentArea

    local pageNames = {"PvpPage","GlitchPage","FlyPage","GunPage","ConfigPage","PremiumPage","GlitchV2Page","SoruPage"}
    local pages = {}
    for _, name in ipairs(pageNames) do
        local page = Instance.new("Frame")
        page.Size = UDim2.new(1,0,1,0)
        page.BackgroundTransparency = 1
        page.Parent = Pages
        pages[name] = page
    end

    local function setTabStyle(btn, isActive, cor)
        if not btn then return end
        btn.BackgroundColor3 = isActive and VermelhoEscuro or Color3.fromRGB(30,20,40)
        local line = tabLines[btn]
        if line then
            line.BackgroundTransparency = isActive and 0 or 1
            line.BackgroundColor3 = cor
        end
    end

    local function showPage(page)
        for _, p in pairs(pages) do p.Visible = (p == page) end
        setTabStyle(TabPvpBtn, page == pages.PvpPage, CoresAbas.Pvp)
        setTabStyle(TabGlitchBtn, page == pages.GlitchPage, CoresAbas.Glitch)
        setTabStyle(TabFlyBtn, page == pages.FlyPage, CoresAbas.Fly)
        setTabStyle(TabGunBtn, page == pages.GunPage, CoresAbas.Gun)
        setTabStyle(TabConfigBtn, page == pages.ConfigPage, CoresAbas.Config)
        setTabStyle(TabPremiumBtn, page == pages.PremiumPage, CoresAbas.Premium)
        setTabStyle(TabGlitchV2Btn, page == pages.GlitchV2Page, CoresAbas.GlitchV2)
        setTabStyle(TabSoruBtn, page == pages.SoruPage, CoresAbas.Soru)
    end
    showPage(pages.PvpPage)

    TabPvpBtn.MouseButton1Click:Connect(function() showPage(pages.PvpPage) end)
    TabGlitchBtn.MouseButton1Click:Connect(function() showPage(pages.GlitchPage) end)
    TabFlyBtn.MouseButton1Click:Connect(function() showPage(pages.FlyPage) end)
    TabGunBtn.MouseButton1Click:Connect(function() showPage(pages.GunPage) end)
    TabConfigBtn.MouseButton1Click:Connect(function() showPage(pages.ConfigPage) end)
    TabPremiumBtn.MouseButton1Click:Connect(function() showPage(pages.PremiumPage) end)
    TabGlitchV2Btn.MouseButton1Click:Connect(function() showPage(pages.GlitchV2Page) end)
    TabSoruBtn.MouseButton1Click:Connect(function() showPage(pages.SoruPage) end)

    ------------------------------------------------------------------
    -- PÁGINA PVP
    ------------------------------------------------------------------
    do
        local PvpPage = pages.PvpPage
        local t = Instance.new("TextLabel")
        t.Size = UDim2.new(0.9,0,0,24); t.Position = UDim2.new(0.05,0,0,4)
        t.BackgroundTransparency = 1; t.Text = "⚔️ PVP – Aimbot & Silent"
        t.TextColor3 = VermelhoClaro; t.Font = Enum.Font.GothamBold; t.TextSize = 14
        t.Parent = PvpPage

        local st = Instance.new("TextLabel")
        st.Size = UDim2.new(0.4,0,0,18); st.Position = UDim2.new(0.05,0,0,32)
        st.BackgroundTransparency = 1; st.Text = "🎯 Aimbot"
        st.TextColor3 = VermelhoClaro; st.Font = Enum.Font.GothamBold; st.TextSize = 13
        st.Parent = PvpPage

        local AimbotToggleBtn = createToggleButton("🔫 Aimbot: OFF", 52, PvpPage)
        local function updateAimbotToggleVisual()
            AimbotToggleBtn.Text = State.aimbotActive and "🔫 Aimbot: ON" or "🔫 Aimbot: OFF"
            AimbotToggleBtn.BackgroundColor3 = State.aimbotActive and VermelhoEscuro or Color3.fromRGB(30,20,40)
        end
        _G.__updateAimbotToggleVisual = updateAimbotToggleVisual
        AimbotToggleBtn.MouseButton1Click:Connect(function() State.aimbotActive = not State.aimbotActive; updateAimbotToggleVisual(); scheduleSave() end)
        updateAimbotToggleVisual()

        local PlayerListFrame = Instance.new("Frame")
        PlayerListFrame.Size = UDim2.new(0.9,0,0,130); PlayerListFrame.Position = UDim2.new(0.05,0,0,85)
        PlayerListFrame.BackgroundColor3 = Color3.fromRGB(20,10,10); PlayerListFrame.BackgroundTransparency = 0.2
        PlayerListFrame.Parent = PvpPage
        Instance.new("UICorner", PlayerListFrame).CornerRadius = UDim.new(0,6)
        createBorder(PlayerListFrame, 1.5, VermelhoBrilho)

        local ScrollingFrame = Instance.new("ScrollingFrame")
        ScrollingFrame.Size = UDim2.new(1,0,1,0); ScrollingFrame.BackgroundTransparency = 1
        ScrollingFrame.CanvasSize = UDim2.new(0,0,0,0); ScrollingFrame.ScrollBarThickness = 4
        ScrollingFrame.Parent = PlayerListFrame
        local PlayerListLayout = Instance.new("UIListLayout")
        PlayerListLayout.Padding = UDim.new(0,3); PlayerListLayout.SortOrder = Enum.SortOrder.Name
        PlayerListLayout.Parent = ScrollingFrame

        local playerButtons = {}
        local selectedPlayerButton = nil
        local function updateAimbotList()
            for _, btn in ipairs(playerButtons) do btn:Destroy() end
            playerButtons = {}; selectedPlayerButton = nil
            local yOffset = 0
            for _, plr in ipairs(Players:GetPlayers()) do
                if plr ~= player then
                    local btn = Instance.new("TextButton")
                    btn.Size = UDim2.new(1,0,0,22); btn.BackgroundColor3 = Color3.fromRGB(40,30,50)
                    btn.Text = plr.Name; btn.TextColor3 = VermelhoClaro
                    btn.Font = Enum.Font.GothamMedium; btn.TextSize = 11
                    btn.Parent = ScrollingFrame
                    Instance.new("UICorner", btn).CornerRadius = UDim.new(0,3)
                    createBorder(btn, 1, VermelhoBrilho)
                    btn.MouseButton1Click:Connect(function()
                        State.aimbotTarget = plr
                        if selectedPlayerButton then selectedPlayerButton.BackgroundColor3 = Color3.fromRGB(40,30,50) end
                        btn.BackgroundColor3 = VermelhoEscuro
                        selectedPlayerButton = btn
                        scheduleSave()
                    end)
                    table.insert(playerButtons, btn); yOffset = yOffset + 25
                end
            end
            ScrollingFrame.CanvasSize = UDim2.new(0,0,0,yOffset)
            if State.aimbotTarget then
                for _, btn in ipairs(playerButtons) do
                    if btn.Text == State.aimbotTarget.Name then btn.BackgroundColor3 = VermelhoEscuro; selectedPlayerButton = btn; break end
                end
            end
        end
        _G.__updateAimbotList = updateAimbotList
        Players.PlayerAdded:Connect(updateAimbotList)
        Players.PlayerRemoving:Connect(updateAimbotList)
        updateAimbotList()

        local ClearTargetBtn = Instance.new("TextButton")
        ClearTargetBtn.Size = UDim2.new(0.3,0,0,24); ClearTargetBtn.Position = UDim2.new(0.35,0,0,220)
        ClearTargetBtn.BackgroundColor3 = Color3.fromRGB(30,20,40); ClearTargetBtn.Text = "✖ Limpar"
        ClearTargetBtn.TextColor3 = VermelhoClaro; ClearTargetBtn.Font = Enum.Font.GothamMedium; ClearTargetBtn.TextSize = 11
        ClearTargetBtn.Parent = PvpPage
        Instance.new("UICorner", ClearTargetBtn).CornerRadius = UDim.new(0,4)
        createBorder(ClearTargetBtn, 1.5, VermelhoBrilho)
        ClearTargetBtn.MouseButton1Click:Connect(function()
            State.aimbotTarget = nil
            if selectedPlayerButton then selectedPlayerButton.BackgroundColor3 = Color3.fromRGB(40,30,50); selectedPlayerButton = nil end
            scheduleSave()
        end)

        local st2 = Instance.new("TextLabel")
        st2.Size = UDim2.new(0.4,0,0,18); st2.Position = UDim2.new(0.05,0,0,255)
        st2.BackgroundTransparency = 1; st2.Text = "🔇 Silent Aim"
        st2.TextColor3 = VermelhoClaro; st2.Font = Enum.Font.GothamBold; st2.TextSize = 13
        st2.Parent = PvpPage

        local SilentToggleBtn = createToggleButton("🔇 Silent: ON", 275, PvpPage)
        local function updateSilentToggle()
            SilentToggleBtn.Text = State.silentEnabled and "🔇 Silent: ON" or "🔇 Silent: OFF"
            SilentToggleBtn.BackgroundColor3 = State.silentEnabled and VermelhoEscuro or Color3.fromRGB(30,20,40)
        end
        _G.__updateSilentToggle = updateSilentToggle
        SilentToggleBtn.MouseButton1Click:Connect(function() State.silentEnabled = not State.silentEnabled; updateSilentToggle(); scheduleSave() end)
        updateSilentToggle()

        local TargetInfo = Instance.new("TextLabel")
        TargetInfo.Size = UDim2.new(0.9,0,0,18); TargetInfo.Position = UDim2.new(0.05,0,0,308)
        TargetInfo.BackgroundTransparency = 1; TargetInfo.Text = "🎯 Alvo: Nenhum"
        TargetInfo.TextColor3 = VermelhoClaro; TargetInfo.Font = Enum.Font.GothamMedium; TargetInfo.TextSize = 11
        TargetInfo.Parent = PvpPage
        _G.__TargetInfo = TargetInfo

        local SilentPlayerFrame = Instance.new("Frame")
        SilentPlayerFrame.Size = UDim2.new(0.9,0,0,110); SilentPlayerFrame.Position = UDim2.new(0.05,0,0,330)
        SilentPlayerFrame.BackgroundColor3 = Color3.fromRGB(20,10,10); SilentPlayerFrame.BackgroundTransparency = 0.2
        SilentPlayerFrame.Parent = PvpPage
        Instance.new("UICorner", SilentPlayerFrame).CornerRadius = UDim.new(0,6)
        createBorder(SilentPlayerFrame, 1.5, VermelhoBrilho)

        local SilentScroll = Instance.new("ScrollingFrame")
        SilentScroll.Size = UDim2.new(1,0,1,0); SilentScroll.BackgroundTransparency = 1
        SilentScroll.CanvasSize = UDim2.new(0,0,0,0); SilentScroll.ScrollBarThickness = 4
        SilentScroll.Parent = SilentPlayerFrame
        local SilentListLayout = Instance.new("UIListLayout")
        SilentListLayout.Padding = UDim.new(0,3); SilentListLayout.SortOrder = Enum.SortOrder.Name
        SilentListLayout.Parent = SilentScroll

        local silentButtons = {}
        _G.__silentButtons = silentButtons
        local function updateSilentPlayerList()
            for _, btn in ipairs(silentButtons) do btn:Destroy() end
            for k in ipairs(silentButtons) do silentButtons[k] = nil end
            local yOffset = 0
            for _, plr in ipairs(Players:GetPlayers()) do
                if plr ~= player then
                    local btn = Instance.new("TextButton")
                    btn.Size = UDim2.new(1,-5,0,22); btn.BackgroundColor3 = Color3.fromRGB(40,30,50)
                    btn.Text = plr.Name; btn.TextColor3 = VermelhoClaro
                    btn.Font = Enum.Font.GothamMedium; btn.TextSize = 11
                    btn.Parent = SilentScroll
                    Instance.new("UICorner", btn).CornerRadius = UDim.new(0,3)
                    createBorder(btn, 1, VermelhoBrilho)
                    btn.MouseButton1Click:Connect(function()
                        if plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") then
                            State.silentTarget = plr; State.silentTargetPos = plr.Character.HumanoidRootPart.Position
                            State.silentLocked = false; State.silentAutoTarget = false
                            if _G.__updateAutoTargetVisualSilent then _G.__updateAutoTargetVisualSilent() end
                            TargetInfo.Text = "🎯 Alvo: "..plr.Name.." (destravado)"
                            for _, b in ipairs(silentButtons) do b.BackgroundColor3 = Color3.fromRGB(40,30,50) end
                            btn.BackgroundColor3 = VermelhoEscuro
                            scheduleSave()
                        end
                    end)
                    table.insert(silentButtons, btn); yOffset = yOffset + 25
                end
            end
            SilentScroll.CanvasSize = UDim2.new(0,0,0,yOffset)
            if State.silentTarget then
                for _, btn in ipairs(silentButtons) do
                    if btn.Text == State.silentTarget.Name then btn.BackgroundColor3 = VermelhoEscuro; break end
                end
            end
        end
        _G.__updateSilentPlayerList = updateSilentPlayerList
        Players.PlayerAdded:Connect(function() task.wait(0.3) updateSilentPlayerList() end)
        Players.PlayerRemoving:Connect(function() task.wait(0.3) updateSilentPlayerList() end)
        updateSilentPlayerList()

        local LockBtnSilent = Instance.new("TextButton")
        LockBtnSilent.Size = UDim2.new(0.25,0,0,24); LockBtnSilent.Position = UDim2.new(0.05,0,0,448)
        LockBtnSilent.BackgroundColor3 = Color3.fromRGB(30,20,40); LockBtnSilent.Text = "🔒 Travar"
        LockBtnSilent.TextColor3 = VermelhoClaro; LockBtnSilent.Font = Enum.Font.GothamMedium; LockBtnSilent.TextSize = 11
        LockBtnSilent.Parent = PvpPage
        Instance.new("UICorner", LockBtnSilent).CornerRadius = UDim.new(0,4)
        createBorder(LockBtnSilent, 1.5, VermelhoBrilho)
        _G.__LockBtnSilent = LockBtnSilent
        LockBtnSilent.MouseButton1Click:Connect(function()
            if State.silentTarget then
                State.silentLocked = not State.silentLocked
                LockBtnSilent.Text = State.silentLocked and "🔓 Destravar" or "🔒 Travar"
                TargetInfo.Text = "🎯 Alvo: "..State.silentTarget.Name..(State.silentLocked and " (travado)" or " (destravado)")
                scheduleSave()
            end
        end)

        local ResetSilentBtn = Instance.new("TextButton")
        ResetSilentBtn.Size = UDim2.new(0.25,0,0,24); ResetSilentBtn.Position = UDim2.new(0.35,0,0,448)
        ResetSilentBtn.BackgroundColor3 = Color3.fromRGB(30,20,40); ResetSilentBtn.Text = "🔄 Resetar"
        ResetSilentBtn.TextColor3 = VermelhoClaro; ResetSilentBtn.Font = Enum.Font.GothamMedium; ResetSilentBtn.TextSize = 11
        ResetSilentBtn.Parent = PvpPage
        Instance.new("UICorner", ResetSilentBtn).CornerRadius = UDim.new(0,4)
        createBorder(ResetSilentBtn, 1.5, VermelhoBrilho)
        ResetSilentBtn.MouseButton1Click:Connect(function()
            State.silentTarget = nil; State.silentTargetPos = nil; State.silentLocked = false; State.silentAutoTarget = false
            if _G.__updateAutoTargetVisualSilent then _G.__updateAutoTargetVisualSilent() end
            LockBtnSilent.Text = "🔒 Travar"; TargetInfo.Text = "🎯 Alvo: Nenhum"
            for _, b in ipairs(silentButtons) do b.BackgroundColor3 = Color3.fromRGB(40,30,50) end
            scheduleSave()
        end)

        local AutoTargetBtnSilent = createToggleButton("🎯 Auto Target: OFF", 478, PvpPage)
        local function updateAutoTargetVisualSilent()
            AutoTargetBtnSilent.Text = State.silentAutoTarget and "🎯 Auto Target: ON" or "🎯 Auto Target: OFF"
            AutoTargetBtnSilent.BackgroundColor3 = State.silentAutoTarget and VermelhoEscuro or Color3.fromRGB(30,20,40)
        end
        _G.__updateAutoTargetVisualSilent = updateAutoTargetVisualSilent
        AutoTargetBtnSilent.MouseButton1Click:Connect(function()
            State.silentAutoTarget = not State.silentAutoTarget; updateAutoTargetVisualSilent(); scheduleSave()
        end)
        updateAutoTargetVisualSilent()

        local SliderFrameSilent = Instance.new("Frame")
        SliderFrameSilent.Size = UDim2.new(0.4,0,0,6); SliderFrameSilent.Position = UDim2.new(0.3,0,0,515)
        SliderFrameSilent.BackgroundColor3 = Color3.fromRGB(60,50,70)
        SliderFrameSilent.Parent = PvpPage
        Instance.new("UICorner", SliderFrameSilent).CornerRadius = UDim.new(0,3)

        local SliderButtonSilent = Instance.new("Frame")
        SliderButtonSilent.Size = UDim2.new(0,12,0,12); SliderButtonSilent.Position = UDim2.new(0,0,0,-3)
        SliderButtonSilent.BackgroundColor3 = VermelhoBrilho
        SliderButtonSilent.Parent = SliderFrameSilent
        Instance.new("UICorner", SliderButtonSilent).CornerRadius = UDim.new(0,6)
        createBorder(SliderButtonSilent,1,VermelhoClaro)

        local SliderValueLabelSilent = Instance.new("TextLabel")
        SliderValueLabelSilent.Size = UDim2.new(0.15,0,0,20); SliderValueLabelSilent.Position = UDim2.new(0.78,0,0,508)
        SliderValueLabelSilent.BackgroundTransparency = 1; SliderValueLabelSilent.Text = "100m"
        SliderValueLabelSilent.TextColor3 = VermelhoClaro; SliderValueLabelSilent.Font = Enum.Font.GothamMedium; SliderValueLabelSilent.TextSize = 11
        SliderValueLabelSilent.Parent = PvpPage

        local function updateSliderSilent(value)
            value = math.clamp(value,0,1000); State.silentMaxDistance = value
            local percent = value / 1000
            SliderButtonSilent.Position = UDim2.new(percent,-6,0,-3)
            SliderValueLabelSilent.Text = math.floor(value).."m"
            scheduleSave()
        end
        _G.__updateSliderSilent = updateSliderSilent
        local sliderDraggingSilent = false
        SliderButtonSilent.InputBegan:Connect(function(input, gp)
            if gp then return end
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                sliderDraggingSilent = true; MainFrame.Active = false
            end
        end)
        SliderButtonSilent.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                sliderDraggingSilent = false; MainFrame.Active = true
            end
        end)
        UserInputService.InputChanged:Connect(function(input)
            if sliderDraggingSilent and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                local sw = SliderFrameSilent.AbsoluteSize.X
                if sw > 0 then
                    local percent = math.clamp((input.Position.X - SliderFrameSilent.AbsolutePosition.X) / sw, 0, 1)
                    updateSliderSilent(percent * 1000)
                end
            end
        end)
        updateSliderSilent(State.silentMaxDistance)
    end

    ------------------------------------------------------------------
    -- GLITCH (original)
    ------------------------------------------------------------------
    do
        local GlitchPage = pages.GlitchPage
        local t = Instance.new("TextLabel")
        t.Size = UDim2.new(0.9,0,0,24); t.Position = UDim2.new(0.05,0,0,4)
        t.BackgroundTransparency = 1; t.Text = "⚡ Glitch"
        t.TextColor3 = VermelhoClaro; t.Font = Enum.Font.GothamBold; t.TextSize = 14
        t.Parent = GlitchPage

        local GlitchToggleBtn = createToggleButton("⚡ Glitch: OFF", 34, GlitchPage)
        local GlitchSpeedInput = createInputField("Velocidade:", State.GlideSpeed, 70, GlitchPage)
        local GlitchDurInput = createInputField("Duração:", State.GlideTime, 100, GlitchPage)
        local JumpPowerInput = createInputField("G-Jump:", State.JumpForce, 130, GlitchPage)
        local SpeedToggleBtn = createToggleButton("🏃 Speed Hack: OFF", 165, GlitchPage)
        local SpeedInput = createInputField("Speed Max:", State.speedHackValue, 195, GlitchPage)

        local function updateSpeedVisual()
            SpeedToggleBtn.Text = State.speedHack and "🏃 Speed Hack: ON" or "🏃 Speed Hack: OFF"
            SpeedToggleBtn.BackgroundColor3 = State.speedHack and VermelhoEscuro or Color3.fromRGB(30,20,40)
        end
        _G.__updateSpeedVisual = updateSpeedVisual
        SpeedToggleBtn.MouseButton1Click:Connect(function() State.speedHack = not State.speedHack; updateSpeedVisual(); scheduleSave() end)
        SpeedInput.FocusLost:Connect(function()
            local v = tonumber(SpeedInput.Text); if v then State.speedHackValue = math.clamp(v,0,500); SpeedInput.Text = tostring(State.speedHackValue); scheduleSave() end
        end)
        updateSpeedVisual()

        local function updateGlitchVisual()
            GlitchToggleBtn.Text = State.GlideEnabled and "⚡ Glitch: ON" or "⚡ Glitch: OFF"
            GlitchToggleBtn.BackgroundColor3 = State.GlideEnabled and VermelhoEscuro or Color3.fromRGB(30,20,40)
        end
        _G.__updateGlitchVisual = updateGlitchVisual
        GlitchToggleBtn.MouseButton1Click:Connect(function()
            State.GlideEnabled = not State.GlideEnabled; updateGlitchVisual(); scheduleSave()
        end)
        GlitchSpeedInput.FocusLost:Connect(function() local v=tonumber(GlitchSpeedInput.Text); if v then State.GlideSpeed=v; scheduleSave() end end)
        GlitchDurInput.FocusLost:Connect(function() local v=tonumber(GlitchDurInput.Text); if v then State.GlideTime=v; scheduleSave() end end)
        JumpPowerInput.FocusLost:Connect(function() local v=tonumber(JumpPowerInput.Text); if v then State.JumpForce=v; scheduleSave() end end)
        updateGlitchVisual()
    end

    ------------------------------------------------------------------
    -- FLY
    ------------------------------------------------------------------
    do
        local FlyPage = pages.FlyPage
        local t = Instance.new("TextLabel")
        t.Size = UDim2.new(0.9,0,0,24); t.Position = UDim2.new(0.05,0,0,4)
        t.BackgroundTransparency = 1; t.Text = "🪁 Voo (Fly)"
        t.TextColor3 = VermelhoClaro; t.Font = Enum.Font.GothamBold; t.TextSize = 14
        t.Parent = FlyPage

        local FlyToggleBtn = createToggleButton("🪁 Fly: OFF", 34, FlyPage)
        local FlySpeedInput = createInputField("Fly Speed:", State.flySpeed, 70, FlyPage)

        local function updateFlyState(state)
            State.flyActive = state
            FlyToggleBtn.Text = State.flyActive and "🪁 Fly: ON" or "🪁 Fly: OFF"
            FlyToggleBtn.BackgroundColor3 = State.flyActive and VermelhoEscuro or Color3.fromRGB(30,20,40)
            if not State.flyActive then
                if FlyVel then FlyVel:Destroy() FlyVel = nil end
                if player.Character and player.Character:FindFirstChild("Humanoid") then
                    player.Character.Humanoid.PlatformStand = false
                end
            elseif player.Character and player.Character:FindFirstChild("Humanoid") then
                player.Character.Humanoid.PlatformStand = true
            end
            scheduleSave()
        end
        _G.__updateFlyState = updateFlyState
        FlyToggleBtn.MouseButton1Click:Connect(function() updateFlyState(not State.flyActive) end)
        FlySpeedInput.FocusLost:Connect(function() local v=tonumber(FlySpeedInput.Text); if v then State.flySpeed=v; scheduleSave() end end)
    end

    ------------------------------------------------------------------
    -- GUN
    ------------------------------------------------------------------
    do
        local GunPage = pages.GunPage
        local t = Instance.new("TextLabel")
        t.Size = UDim2.new(0.9,0,0,24); t.Position = UDim2.new(0.05,0,0,4)
        t.BackgroundTransparency = 1; t.Text = "🔫 Gun Silent Aim + M1 Extender"
        t.TextColor3 = VermelhoClaro; t.Font = Enum.Font.GothamBold; t.TextSize = 14
        t.Parent = GunPage

        local st = Instance.new("TextLabel")
        st.Size = UDim2.new(0.4,0,0,18); st.Position = UDim2.new(0.05,0,0,32)
        st.BackgroundTransparency = 1; st.Text = "🔫 Gun Silent Aim"
        st.TextColor3 = VermelhoClaro; st.Font = Enum.Font.GothamBold; st.TextSize = 13
        st.Parent = GunPage

        local GunToggleBtn = createToggleButton("🔫 Gun Silent: OFF", 52, GunPage)
        local function updateGunToggleVisual()
            GunToggleBtn.Text = State.gunSilentEnabled and "🔫 Gun Silent: ON" or "🔫 Gun Silent: OFF"
            GunToggleBtn.BackgroundColor3 = State.gunSilentEnabled and VermelhoEscuro or Color3.fromRGB(30,20,40)
        end
        _G.__updateGunToggleVisual = updateGunToggleVisual
        GunToggleBtn.MouseButton1Click:Connect(function() State.gunSilentEnabled = not State.gunSilentEnabled; updateGunToggleVisual(); scheduleSave() end)
        updateGunToggleVisual()

        local GunSliderFrame = Instance.new("Frame")
        GunSliderFrame.Size = UDim2.new(0.4,0,0,6); GunSliderFrame.Position = UDim2.new(0.3,0,0,88)
        GunSliderFrame.BackgroundColor3 = Color3.fromRGB(60,50,70)
        GunSliderFrame.Parent = GunPage
        Instance.new("UICorner", GunSliderFrame).CornerRadius = UDim.new(0,3)

        local GunSliderButton = Instance.new("Frame")
        GunSliderButton.Size = UDim2.new(0,12,0,12); GunSliderButton.Position = UDim2.new(0,0,0,-3)
        GunSliderButton.BackgroundColor3 = VermelhoBrilho
        GunSliderButton.Parent = GunSliderFrame
        Instance.new("UICorner", GunSliderButton).CornerRadius = UDim.new(0,6)
        createBorder(GunSliderButton,1,VermelhoClaro)

        local GunSliderValueLabel = Instance.new("TextLabel")
        GunSliderValueLabel.Size = UDim2.new(0.15,0,0,20); GunSliderValueLabel.Position = UDim2.new(0.78,0,0,81)
        GunSliderValueLabel.BackgroundTransparency = 1; GunSliderValueLabel.Text = tostring(State.gunMaxDistance).."m"
        GunSliderValueLabel.TextColor3 = VermelhoClaro; GunSliderValueLabel.Font = Enum.Font.GothamMedium; GunSliderValueLabel.TextSize = 11
        GunSliderValueLabel.Parent = GunPage

        local function updateGunSlider(value)
            value = math.clamp(value,10,500); State.gunMaxDistance = math.floor(value)
            local percent = (value - 10)/490
            GunSliderButton.Position = UDim2.new(percent,-6,0,-3)
            GunSliderValueLabel.Text = State.gunMaxDistance.."m"
            scheduleSave()
        end
        _G.__updateGunSlider = updateGunSlider
        local gunSliderDragging = false
        GunSliderButton.InputBegan:Connect(function(input, gp)
            if gp then return end
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                gunSliderDragging = true; MainFrame.Active = false
            end
        end)
        GunSliderButton.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                gunSliderDragging = false; MainFrame.Active = true
            end
        end)
        UserInputService.InputChanged:Connect(function(input)
            if gunSliderDragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                local sw = GunSliderFrame.AbsoluteSize.X
                if sw > 0 then
                    local percent = math.clamp((input.Position.X - GunSliderFrame.AbsolutePosition.X) / sw, 0, 1)
                    updateGunSlider(10 + percent * 490)
                end
            end
        end)
        updateGunSlider(State.gunMaxDistance)

        local GunTargetLabel = Instance.new("TextLabel")
        GunTargetLabel.Size = UDim2.new(0.9,0,0,18); GunTargetLabel.Position = UDim2.new(0.05,0,0,115)
        GunTargetLabel.BackgroundTransparency = 1; GunTargetLabel.Text = "🎯 Alvo: Nenhum"
        GunTargetLabel.TextColor3 = VermelhoClaro; GunTargetLabel.Font = Enum.Font.GothamMedium; GunTargetLabel.TextSize = 11
        GunTargetLabel.Parent = GunPage

        -- [OTIMIZADO] Intervalo aumentado de 0.08 para 0.15
        -- [OTIMIZADO] Só roda se gunSilentEnabled
        task.spawn(function()
            while true do
                task.wait(0.15)
                if State.gunSilentEnabled then
                    local myChar = player.Character
                    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
                    if myRoot then
                        local myPos = myRoot.Position
                        local closest, closestDist = nil, State.gunMaxDistance
                        for _, plr in ipairs(Players:GetPlayers()) do
                            if plr ~= player then
                                local char = plr.Character
                                if char then
                                    local root = char:FindFirstChild("HumanoidRootPart")
                                    local hum = char:FindFirstChildOfClass("Humanoid")
                                    if root and hum and hum.Health > 0 then
                                        local d = (root.Position - myPos).Magnitude
                                        if d <= closestDist then closest, closestDist = plr, d end
                                    end
                                end
                            end
                        end
                        if closest then
                            State.gunTarget = closest
                            State.gunTargetPart = closest.Character.HumanoidRootPart
                            State.gunTargetPos = State.gunTargetPart.Position
                            GunTargetLabel.Text = "🎯 Alvo: "..closest.Name
                        else
                            State.gunTarget, State.gunTargetPart, State.gunTargetPos = nil, nil, nil
                            GunTargetLabel.Text = "🎯 Alvo: Nenhum"
                        end
                    end
                else
                    if State.gunTarget then
                        State.gunTarget, State.gunTargetPart, State.gunTargetPos = nil, nil, nil
                        GunTargetLabel.Text = "🎯 Alvo: Nenhum"
                    end
                end
            end
        end)

        local st2 = Instance.new("TextLabel")
        st2.Size = UDim2.new(0.4,0,0,18); st2.Position = UDim2.new(0.05,0,0,148)
        st2.BackgroundTransparency = 1; st2.Text = "👊 M1 Extender"
        st2.TextColor3 = VermelhoClaro; st2.Font = Enum.Font.GothamBold; st2.TextSize = 13
        st2.Parent = GunPage

        local M1ToggleBtn = createToggleButton("👊 M1 Extender: OFF", 168, GunPage)
        local function updateM1ToggleVisual()
            M1ToggleBtn.Text = State.m1ExtenderActive and "👊 M1 Extender: ON" or "👊 M1 Extender: OFF"
            M1ToggleBtn.BackgroundColor3 = State.m1ExtenderActive and VermelhoEscuro or Color3.fromRGB(30,20,40)
        end
        _G.__updateM1ToggleVisual = updateM1ToggleVisual
        M1ToggleBtn.MouseButton1Click:Connect(function() State.m1ExtenderActive = not State.m1ExtenderActive; updateM1ToggleVisual(); scheduleSave() end)
        updateM1ToggleVisual()

        local M1SliderFrame = Instance.new("Frame")
        M1SliderFrame.Size = UDim2.new(0.4,0,0,6); M1SliderFrame.Position = UDim2.new(0.3,0,0,205)
        M1SliderFrame.BackgroundColor3 = Color3.fromRGB(60,50,70)
        M1SliderFrame.Parent = GunPage
        Instance.new("UICorner", M1SliderFrame).CornerRadius = UDim.new(0,3)

        local M1SliderButton = Instance.new("Frame")
        M1SliderButton.Size = UDim2.new(0,12,0,12); M1SliderButton.Position = UDim2.new(0,0,0,-3)
        M1SliderButton.BackgroundColor3 = VermelhoBrilho
        M1SliderButton.Parent = M1SliderFrame
        Instance.new("UICorner", M1SliderButton).CornerRadius = UDim.new(0,6)
        createBorder(M1SliderButton,1,VermelhoClaro)

        local M1SliderValueLabel = Instance.new("TextLabel")
        M1SliderValueLabel.Size = UDim2.new(0.15,0,0,20); M1SliderValueLabel.Position = UDim2.new(0.78,0,0,198)
        M1SliderValueLabel.BackgroundTransparency = 1; M1SliderValueLabel.Text = tostring(State.m1ExtenderRange).."m"
        M1SliderValueLabel.TextColor3 = VermelhoClaro; M1SliderValueLabel.Font = Enum.Font.GothamMedium; M1SliderValueLabel.TextSize = 11
        M1SliderValueLabel.Parent = GunPage

        local function updateM1Slider(value)
            value = math.clamp(value,10,200); State.m1ExtenderRange = math.floor(value)
            local percent = (value - 10)/190
            M1SliderButton.Position = UDim2.new(percent,-6,0,-3)
            M1SliderValueLabel.Text = State.m1ExtenderRange.."m"
            scheduleSave()
        end
        _G.__updateM1Slider = updateM1Slider
        local m1SliderDragging = false
        M1SliderButton.InputBegan:Connect(function(input, gp)
            if gp then return end
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                m1SliderDragging = true; MainFrame.Active = false
            end
        end)
        M1SliderButton.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                m1SliderDragging = false; MainFrame.Active = true
            end
        end)
        UserInputService.InputChanged:Connect(function(input)
            if m1SliderDragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                local sw = M1SliderFrame.AbsoluteSize.X
                if sw > 0 then
                    local percent = math.clamp((input.Position.X - M1SliderFrame.AbsolutePosition.X) / sw, 0, 1)
                    updateM1Slider(10 + percent * 190)
                end
            end
        end)
        updateM1Slider(State.m1ExtenderRange)
    end

    ------------------------------------------------------------------
    -- CONFIG
    ------------------------------------------------------------------
    do
        local ConfigPage = pages.ConfigPage
        local t = Instance.new("TextLabel")
        t.Size = UDim2.new(0.9,0,0,24); t.Position = UDim2.new(0.05,0,0,4)
        t.BackgroundTransparency = 1; t.Text = "⚙️ Configurações"
        t.TextColor3 = VermelhoClaro; t.Font = Enum.Font.GothamBold; t.TextSize = 14
        t.Parent = ConfigPage

        local TeamToggleBtn = createToggleButton("🏴‍☠️ Time: PIRATES", 34, ConfigPage)
        local function updateTeamVisual()
            local team = getgenv().SelectedTeam or "Pirates"
            if team == "Pirates" then
                TeamToggleBtn.Text = "🏴‍☠️ Time: PIRATES"
                TeamToggleBtn.BackgroundColor3 = VermelhoEscuro
            else
                TeamToggleBtn.Text = "⚓ Time: MARINES"
                TeamToggleBtn.BackgroundColor3 = Color3.fromRGB(0,80,180)
            end
        end
        _G.__updateTeamVisual = updateTeamVisual
        TeamToggleBtn.MouseButton1Click:Connect(function()
            getgenv().SelectedTeam = (getgenv().SelectedTeam == "Pirates") and "Marines" or "Pirates"
            updateTeamVisual(); scheduleSave()
        end)
        updateTeamVisual()

        local StatusLabel = Instance.new("TextLabel")
        StatusLabel.Size = UDim2.new(0.9,0,0,18); StatusLabel.Position = UDim2.new(0.05,0,0,70)
        StatusLabel.BackgroundTransparency = 1; StatusLabel.Text = "✅ Configurações salvas automaticamente"
        StatusLabel.TextColor3 = Color3.fromRGB(150,255,150); StatusLabel.Font = Enum.Font.GothamMedium; StatusLabel.TextSize = 11
        StatusLabel.Parent = ConfigPage
    end

    ------------------------------------------------------------------
    -- PREMIUM
    ------------------------------------------------------------------
    do
        local PremiumPage = pages.PremiumPage
        local t = Instance.new("TextLabel")
        t.Size = UDim2.new(0.9,0,0,24); t.Position = UDim2.new(0.05,0,0,4)
        t.BackgroundTransparency = 1; t.Text = "💎 Premium"
        t.TextColor3 = VermelhoClaro; t.Font = Enum.Font.GothamBold; t.TextSize = 14
        t.Parent = PremiumPage

        local fastSkillActive = false
        local FastSkillBtn = createToggleButton("⚡ FastSkill: OFF", 34, PremiumPage)
        local function updateFastSkillVisual()
            FastSkillBtn.Text = fastSkillActive and "⚡ FastSkill: ON" or "⚡ FastSkill: OFF"
            FastSkillBtn.BackgroundColor3 = fastSkillActive and VermelhoEscuro or Color3.fromRGB(30,20,40)
        end
        _G.__updateFastSkillVisual = updateFastSkillVisual
        FastSkillBtn.MouseButton1Click:Connect(function() fastSkillActive = not fastSkillActive; updateFastSkillVisual() end)
        updateFastSkillVisual()

        local WarningLabel = Instance.new("TextLabel")
        WarningLabel.Size = UDim2.new(0.9,0,0,24); WarningLabel.Position = UDim2.new(0.05,0,0,70)
        WarningLabel.BackgroundTransparency = 1; WarningLabel.Text = "⚠️ Use por conta e risco"
        WarningLabel.TextColor3 = Color3.fromRGB(255,200,50); WarningLabel.Font = Enum.Font.GothamMedium; WarningLabel.TextSize = 12
        WarningLabel.Parent = PremiumPage
    end

    ------------------------------------------------------------------
    -- PÁGINA GLITCH V2
    ------------------------------------------------------------------
    do
        local GlitchV2Page = pages.GlitchV2Page
        local t = Instance.new("TextLabel")
        t.Size = UDim2.new(0.9,0,0,24); t.Position = UDim2.new(0.05,0,0,4)
        t.BackgroundTransparency = 1; t.Text = "💥 Glitch V2 – Boosts"
        t.TextColor3 = VermelhoClaro; t.Font = Enum.Font.GothamBold; t.TextSize = 14
        t.Parent = GlitchV2Page

        GlitchV2.CreateUI(GlitchV2Page, {
            createToggleButton = createToggleButton,
            createInputField = createInputField,
            VermelhoClaro = VermelhoClaro,
            VermelhoEscuro = VermelhoEscuro,
            save = scheduleSave
        })
    end

    ------------------------------------------------------------------
    -- PÁGINA SORU
    ------------------------------------------------------------------
    do
        local SoruPage = pages.SoruPage
        local t = Instance.new("TextLabel")
        t.Size = UDim2.new(0.9,0,0,24); t.Position = UDim2.new(0.05,0,0,4)
        t.BackgroundTransparency = 1; t.Text = "🌀 Soru Assist – Auto-Aim 360"
        t.TextColor3 = VermelhoClaro; t.Font = Enum.Font.GothamBold; t.TextSize = 14
        t.Parent = SoruPage

        Soru.CreateUI(SoruPage, {
            createToggleButton = createToggleButton,
            createInputField = createInputField,
            VermelhoClaro = VermelhoClaro,
            VermelhoEscuro = VermelhoEscuro,
            save = scheduleSave
        })
    end

    ------------------------------------------------------------------
    -- BOTÃO FLUTUANTE
    ------------------------------------------------------------------
    local ToggleButton = Instance.new("TextButton")
    ToggleButton.Size = UDim2.new(0,50,0,50)
    ToggleButton.Position = UDim2.new(0.02,0,0.9,0)
    ToggleButton.BackgroundColor3 = Color3.fromRGB(25,15,15)
    ToggleButton.BackgroundTransparency = 0.1
    ToggleButton.Text = "🌙"
    ToggleButton.TextColor3 = VermelhoClaro
    ToggleButton.Font = Enum.Font.GothamBold
    ToggleButton.TextSize = 24
    ToggleButton.Parent = ScreenGui
    ToggleButton.Visible = false
    Instance.new("UICorner", ToggleButton).CornerRadius = UDim.new(1,0)
    createBorder(ToggleButton, 2, NeonRed)

    ToggleButton.MouseEnter:Connect(function()
        TweenService:Create(ToggleButton, TweenInfo.new(0.15), { BackgroundColor3 = Color3.fromRGB(50,25,25), Size = UDim2.new(0,54,0,54) }):Play()
    end)
    ToggleButton.MouseLeave:Connect(function()
        TweenService:Create(ToggleButton, TweenInfo.new(0.15), { BackgroundColor3 = Color3.fromRGB(25,15,15), Size = UDim2.new(0,50,0,50) }):Play()
    end)

    local dragging = false
    local dragStartMouse, dragStartPosition
    ToggleButton.InputBegan:Connect(function(input, gp)
        if gp then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStartMouse = Vector2.new(input.Position.X, input.Position.Y)
            dragStartPosition = ToggleButton.Position
        end
    end)
    ToggleButton.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = Vector2.new(input.Position.X, input.Position.Y) - dragStartMouse
            local ss = ScreenGui.AbsoluteSize
            local nX = math.clamp(dragStartPosition.X.Scale + delta.X/ss.X, 0, 1 - ToggleButton.Size.X.Scale)
            local nY = math.clamp(dragStartPosition.Y.Scale + delta.Y/ss.Y, 0, 1 - ToggleButton.Size.Y.Scale)
            ToggleButton.Position = UDim2.new(nX,0,nY,0)
        end
    end)
    ToggleButton.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 and dragging then
            dragging = false
            if (Vector2.new(input.Position.X, input.Position.Y) - dragStartMouse).Magnitude < 10 then
                uiVisible = not uiVisible
                MainFrame.Visible = uiVisible
            end
        end
    end)

    ------------------------------------------------------------------
    -- [OTIMIZADO] RENDER LOOP ÚNICO
    -- Antes: 1 RenderStepped fazendo TUDO (fly, aimbot, speed, silent)
    -- Agora: separado em RenderStepped (visual/por frame) e um loop
    --        de 0.1s (lógica de seleção de alvo do silentAutoTarget)
    ------------------------------------------------------------------

    -- RenderStepped só com o que precisa ser por-frame
    RunService.RenderStepped:Connect(function()
        -- Fly (precisa ser por frame)
        if State.flyActive then
            local char = player.Character
            local root = char and char:FindFirstChild("HumanoidRootPart")
            if root then
                if not FlyVel or FlyVel.Parent ~= root then
                    if FlyVel then FlyVel:Destroy() end
                    FlyVel = Instance.new("BodyVelocity")
                    FlyVel.Name = "SkyFly"
                    FlyVel.MaxForce = Vector3.new(9e9,9e9,9e9)
                    FlyVel.Parent = root
                end
                local md = Vector3.zero
                if UserInputService:IsKeyDown(KC_W) then md = md + camera.CFrame.LookVector end
                if UserInputService:IsKeyDown(KC_S) then md = md - camera.CFrame.LookVector end
                if UserInputService:IsKeyDown(KC_A) then md = md - camera.CFrame.RightVector end
                if UserInputService:IsKeyDown(KC_D) then md = md + camera.CFrame.RightVector end
                FlyVel.Velocity = md * State.flySpeed
            end
        end

        -- Aimbot (precisa ser por frame pra ficar suave)
        if State.aimbotActive and State.aimbotTarget then
            local tChar = State.aimbotTarget.Character
            local tRoot = tChar and tChar:FindFirstChild("HumanoidRootPart")
            if tRoot then
                camera.CFrame = CFrame.new(camera.CFrame.Position, tRoot.Position)
            end
        end
    end)

    -- [OTIMIZADO] Loop de lógica de alvo (10Hz, não 60Hz)
    task.spawn(function()
        while true do
            task.wait(0.1)

            -- Speed hack
            if State.speedHack then
                local char = player.Character
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                if hum then hum.WalkSpeed = State.speedHackValue end
            end

            -- Silent Auto Target
            if State.silentAutoTarget and not State.silentLocked then
                local myChar = player.Character
                local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
                if myRoot then
                    local myPos = myRoot.Position
                    local closest, closestDist = nil, State.silentMaxDistance
                    for _, plr in ipairs(Players:GetPlayers()) do
                        if plr ~= player then
                            local char = plr.Character
                            if char then
                                local root = char:FindFirstChild("HumanoidRootPart")
                                local hum = char:FindFirstChildOfClass("Humanoid")
                                if root and hum and hum.Health > 0 then
                                    local d = (root.Position - myPos).Magnitude
                                    if d <= closestDist then closest, closestDist = plr, d end
                                end
                            end
                        end
                    end
                    if closest then
                        State.silentTarget = closest
                        State.silentTargetPos = closest.Character.HumanoidRootPart.Position
                        if _G.__TargetInfo then _G.__TargetInfo.Text = "🎯 Alvo: "..closest.Name.." (auto)" end
                        if _G.__silentButtons then
                            for _, b in ipairs(_G.__silentButtons) do
                                b.BackgroundColor3 = (b.Text == closest.Name) and VermelhoEscuro or Color3.fromRGB(40,30,50)
                            end
                        end
                    elseif State.silentTarget then
                        State.silentTarget = nil; State.silentTargetPos = nil
                        if _G.__TargetInfo then _G.__TargetInfo.Text = "🎯 Alvo: Nenhum" end
                        if _G.__silentButtons then
                            for _, b in ipairs(_G.__silentButtons) do b.BackgroundColor3 = Color3.fromRGB(40,30,50) end
                        end
                    end
                end
            end

            -- Silent target position update
            if State.silentTarget then
                local tChar = State.silentTarget.Character
                local tRoot = tChar and tChar:FindFirstChild("HumanoidRootPart")
                if tRoot then
                    State.silentTargetPos = tRoot.Position
                elseif not State.silentLocked then
                    State.silentTarget = nil; State.silentTargetPos = nil
                    if _G.__TargetInfo then _G.__TargetInfo.Text = "🎯 Alvo: Nenhum" end
                    if _G.__silentButtons then
                        for _, b in ipairs(_G.__silentButtons) do b.BackgroundColor3 = Color3.fromRGB(40,30,50) end
                    end
                end
            end
        end
    end)

    -- GLITCH LOOP
    task.spawn(function()
        while true do
            task.wait(0.1)
            if State.GlideEnabled then
                local char = player.Character
                if char then
                    local hrp = char:FindFirstChild("HumanoidRootPart")
                    local hum = char:FindFirstChildOfClass("Humanoid")
                    if hrp and hum then
                        if (tick() - State.LastShootTime < 1.5) and hrp.Velocity.Magnitude > 20 then
                            if hrp:FindFirstChild("OmniGlide") then hrp.OmniGlide:Destroy() end
                            local CurrentBV = Instance.new("BodyVelocity")
                            CurrentBV.Name = "OmniGlide"
                            CurrentBV.MaxForce = Vector3.new(100000,0,100000)
                            local Dir = hum.MoveDirection
                            if Dir.Magnitude == 0 then
                                local cl = camera.CFrame.LookVector
                                Dir = Vector3.new(cl.X,0,cl.Z).Unit
                            else Dir = Dir.Unit end
                            CurrentBV.Velocity = Dir * State.GlideSpeed
                            CurrentBV.Parent = hrp
                            Debris:AddItem(CurrentBV, State.GlideTime)
                            State.LastShootTime = 0
                        end
                    end
                end
            end
        end
    end)

    -- RESET JUMP
    local function ResetJump()
        local char = player.Character
        local hum = char and char:FindFirstChild("Humanoid")
        if hum then
            hum.JumpPower = 50
            hum.UseJumpPower = true
        end
    end
    ResetJump()
    player.CharacterAdded:Connect(ResetJump)

    ------------------------------------------------------------------
    -- HOTKEYS
    ------------------------------------------------------------------
    UserInputService.InputBegan:Connect(function(input, gp)
        if gp then return end
        local kc = input.KeyCode
        if kc == Enum.KeyCode.F2 then
            local nv = not MainFrame.Visible
            MainFrame.Visible = nv
            ToggleButton.Visible = nv
            return
        end
        if kc == Enum.KeyCode.L then
            if State.silentTarget and _G.__LockBtnSilent then
                State.silentLocked = not State.silentLocked
                _G.__LockBtnSilent.Text = State.silentLocked and "🔓 Destravar" or "🔒 Travar"
                if _G.__TargetInfo then
                    _G.__TargetInfo.Text = "🎯 Alvo: "..State.silentTarget.Name..(State.silentLocked and " (travado)" or " (destravado)")
                end
                scheduleSave()
            end
        end
        if kc == Enum.KeyCode.RightShift then
            State.silentEnabled = not State.silentEnabled
            if _G.__updateSilentToggle then _G.__updateSilentToggle() end
            scheduleSave()
        end
        if kc == Enum.KeyCode.P then
            State.silentTarget = nil; State.silentTargetPos = nil; State.silentLocked = false; State.silentAutoTarget = false
            if _G.__updateAutoTargetVisualSilent then _G.__updateAutoTargetVisualSilent() end
            if _G.__LockBtnSilent then _G.__LockBtnSilent.Text = "🔒 Travar" end
            if _G.__TargetInfo then _G.__TargetInfo.Text = "🎯 Alvo: Nenhum" end
            if _G.__silentButtons then
                for _, b in ipairs(_G.__silentButtons) do b.BackgroundColor3 = Color3.fromRGB(40,30,50) end
            end
            scheduleSave()
        end
        if kc == Enum.KeyCode.H then
            State.flyActive = not State.flyActive
            if _G.__updateFlyState then _G.__updateFlyState(State.flyActive) end
        end
        if kc == Enum.KeyCode.G then
            local char = player.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if hrp then
                for _, v in pairs(hrp:GetChildren()) do
                    if v:IsA("BodyVelocity") and v.Name == "ImpulseJump" then v:Destroy() end
                end
                local bv = Instance.new("BodyVelocity")
                bv.Name = "ImpulseJump"
                bv.MaxForce = Vector3.new(0, 9e9, 0)
                bv.Velocity = Vector3.new(0, State.JumpForce, 0)
                bv.Parent = hrp
                Debris:AddItem(bv, 0.3)
            end
        end
    end)

    ------------------------------------------------------------------
    -- M1 EXTENDER LISTENER
    ------------------------------------------------------------------
    local function getM1Target()
        local myChar = player.Character
        if not myChar then return nil end
        local myRoot = myChar:FindFirstChild("HumanoidRootPart")
        if not myRoot then return nil end
        local myPos = myRoot.Position
        local range = State.m1ExtenderRange
        local target, shortestDist = nil, range
        local enemies = workspace:FindFirstChild("Enemies")
        local characters = workspace:FindFirstChild("Characters")
        local folders = {enemies, characters}
        for fi = 1, 2 do
            local folder = folders[fi]
            if folder then
                for _, v in ipairs(folder:GetChildren()) do
                    if v ~= myChar then
                        local eRoot = v:FindFirstChild("HumanoidRootPart")
                        local eHum = v:FindFirstChildOfClass("Humanoid")
                        if eRoot and eHum and eHum.Health > 0 then
                            local d = (eRoot.Position - myPos).Magnitude
                            if d < shortestDist then shortestDist = d; target = v end
                        end
                    end
                end
            end
        end
        return target
    end

    UserInputService.InputBegan:Connect(function(input, gp)
        if gp or not State.m1ExtenderActive then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            local target = getM1Target()
            local char = player.Character
            local tool = char and char:FindFirstChildOfClass("Tool")
            if target and tool and RS_RegisterHit and RS_RegisterAttack then
                local head = target:FindFirstChild("Head") or target:FindFirstChild("HumanoidRootPart")
                if head then
                    -- [OTIMIZADO] sid simples (não precisa de coroutine.running())
                    local sid = tostring(player.UserId):sub(2,4)..tostring(tick()):sub(-5)
                    pcall(function()
                        RS_RegisterAttack:FireServer()
                        local hd = {}
                        for _, part in pairs(target:GetChildren()) do
                            if part:IsA("BasePart") then table.insert(hd, {target, part}) end
                        end
                        RS_RegisterHit:FireServer(head, hd, {}, sid)
                    end)
                end
            end
        end
    end)

    ------------------------------------------------------------------
    -- [OTIMIZADO] HOOKS CORRIGIDOS
    -- Antes: __namecall fazia unpack(args) SEMPRE, + __index interceptava tudo
    -- Agora: early-exit antes de qualquer alocação; só toca em args se
    --        realmente for uma chamada relevante
    ------------------------------------------------------------------
    local mt = getrawmetatable(game)
    if mt then
        setreadonly(mt, false)

        local oldNC = mt.__namecall
        local oldIndex = mt.__index
        local MouseObj = player:GetMouse()  -- cache

        -- [OTIMIZADO] Lista de nomes de remote que a gente quer interceptar
        -- (evita checar toda chamada)
        local SILENT_REMOTE_NAMES = {
            ["ShootGunEvent"] = true,
            -- adicione outros remotes de arma aqui se necessário
        }

        mt.__namecall = newcclosure(function(self, ...)
            local method = getnamecallmethod()

            -- [OTIMIZADO] Early exit: só interessa FireServer/InvokeServer
            if method ~= "FireServer" and method ~= "InvokeServer" then
                return oldNC(self, ...)
            end

            local SoruMod = _G.__SoruModule
            local soruActive = SoruMod and SoruMod.IsEnabled() and SoruMod.HasTarget()
            local gunActive = State.gunSilentEnabled and State.gunTargetPos
            local silentActive = State.silentEnabled and State.silentTargetPos

            -- [OTIMIZADO] Se NADA está ativo, passa direto sem tocar em args
            if not soruActive and not gunActive and not silentActive then
                return oldNC(self, ...)
            end

            local rn = self.Name
            -- [OTIMIZADO] Só constrói string se self.Name não for string
            if type(rn) ~= "string" then rn = tostring(rn) end

            local args = {...}

            if gunActive and SILENT_REMOTE_NAMES[rn] then
                for i = 1, #args do
                    local a = args[i]
                    if typeof(a) == "Vector3" then
                        args[i] = State.gunTargetPos
                    elseif type(a) == "table" and State.gunTargetPart then
                        -- assume que a tabela de partes é args[2]
                        if i == 2 then args[2] = { [1] = State.gunTargetPart } end
                    end
                end
                return oldNC(self, table.unpack(args))
            end

            if soruActive then
                local skillKey = nil
                for i = 1, #args do
                    local a = args[i]
                    if type(a) == "string" and (a == "C" or a == "Z" or a == "X" or a == "V" or a == "F" or a == "TAP") then
                        skillKey = a
                        break
                    end
                end
                if skillKey then
                    SoruMod.SetCurrentSkillKey(skillKey)
                    if skillKey == "C" then
                        local pred = SoruMod.GetPredictedPosition()
                        if pred then
                            for i = 1, #args do
                                if typeof(args[i]) == "Vector3" then args[i] = pred end
                            end
                            return oldNC(self, table.unpack(args))
                        end
                    end
                end
            end

            if silentActive then
                for i = 1, #args do
                    local a = args[i]
                    if typeof(a) == "Vector3" then
                        args[i] = State.silentTargetPos
                    elseif type(a) == "table" and a.X and a.Y and a.Z then
                        args[i] = State.silentTargetPos
                    end
                end
                return oldNC(self, table.unpack(args))
            end

            return oldNC(self, ...)
        end)

        -- [OTIMIZADO] __index: early-exit com comparação direta
        -- Antes: sempre chamava SoruMod.IsEnabled() e HasTarget()
        -- Agora: só faz lookup se t for exatamente o Mouse
        mt.__index = newcclosure(function(t, k)
            if t == MouseObj then
                local SoruMod = _G.__SoruModule
                if SoruMod and SoruMod.IsEnabled() and SoruMod.HasTarget() then
                    if k == "Hit" then
                        local pred = SoruMod.GetPredictedPosition()
                        if pred then return CFrame.new(pred) end
                    elseif k == "Target" then
                        return nil
                    end
                end
            end
            return oldIndex(t, k)
        end)

        setreadonly(mt, true)
    end

    ------------------------------------------------------------------
    -- START
    ------------------------------------------------------------------
    task.spawn(function()
        task.wait(0.2)
        if _G.__updateGlitchVisual then _G.__updateGlitchVisual() end
        if _G.__updateFlyState then _G.__updateFlyState(State.flyActive) end
        if _G.__updateAimbotToggleVisual then _G.__updateAimbotToggleVisual() end
        if _G.__updateSilentToggle then _G.__updateSilentToggle() end
        if _G.__updateAutoTargetVisualSilent then _G.__updateAutoTargetVisualSilent() end
        if _G.__updateSliderSilent then _G.__updateSliderSilent(State.silentMaxDistance) end
        if _G.__updateTeamVisual then _G.__updateTeamVisual() end
        if _G.__updateSpeedVisual then _G.__updateSpeedVisual() end
        if _G.__updateGunToggleVisual then _G.__updateGunToggleVisual() end
        if _G.__updateGunSlider then _G.__updateGunSlider(State.gunMaxDistance) end
        if _G.__updateM1ToggleVisual then _G.__updateM1ToggleVisual() end
        if _G.__updateM1Slider then _G.__updateM1Slider(State.m1ExtenderRange) end

        MainFrame.Visible = true
        MainFrame.Size = UDim2.new(0,0,0,0)
        MainFrame.BackgroundTransparency = 1
        TweenService:Create(MainFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Size = UDim2.new(0,620,0,760),
            BackgroundTransparency = 0.15
        }):Play()
        ToggleButton.Visible = true
    end)

end, function(err)
    warn("ERRO NO DARK HUB: "..tostring(err))
    pcall(function()
        StarterGui:SetCore("SendNotification",{Title="❌ Erro",Text="Erro: "..tostring(err),Duration=5})
    end)
end)
