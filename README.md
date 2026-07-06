-- ===== CONFIG =====
local CONFIG = {
    autoCollect = true,
    autoSell    = false,
    autoBuy     = true,
    autoBuyBM   = true,
    interval    = 2,

    buyItems = {
        Farm     = {"Library"},
        House    = {},
        Military = {"MechStation"},
        Decor    = {"WorkerStatue", "SoldierStatue"},
    },

    buyBMItems = {"GemMine", "CloneFacility", "CloneFacilityV2"},
}
-- ==================

local req = request or http_request or (syn and syn.request)
assert(req, "Executor ไม่รองรับ request function")

local function loadUrl(url)
    local res = req({
        Method = "GET",
        Url = url
    })

    assert(res and res.Body, "โหลด URL ไม่ได้: " .. url)
    return loadstring(res.Body)()
end

local Fluent = loadUrl("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua")
local SaveManager = loadUrl("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua")
local InterfaceManager = loadUrl("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua")

local RS = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local lp = Players.LocalPlayer

local GetBridge = require(RS.util.GetBridge)
local BuildingsConfig = require(RS.shared.config.BuildingsConfig)
local ShopsConfig = require(RS.shared.config.ShopsConfig)

local remote = RS:WaitForChild("ncxyzero_bridgenet2-fork@1.1.5"):WaitForChild("dataRemoteEvent")

-- ===== State =====
local autoCollect = CONFIG.autoCollect
local autoSell    = CONFIG.autoSell
local autoBuy     = CONFIG.autoBuy
local autoBuyBM   = CONFIG.autoBuyBM

local selectedItems = {
    Farm = {},
    House = {},
    Military = {},
    Decor = {}
}

for cat, items in pairs(CONFIG.buyItems) do
    for _, name in ipairs(items) do
        if selectedItems[cat] then
            selectedItems[cat][name] = true
        end
    end
end

local selectedBM = {}

for _, name in ipairs(CONFIG.buyBMItems) do
    selectedBM[name] = true
end

-- ===== Helpers =====
local function getMyPlot()
    local map = workspace:FindFirstChild("MilitaryMap")
    if not map then return nil end

    local pp = map:FindFirstChild("PlayerPlots")
    if not pp then return nil end

    for _, plot in ipairs(pp:GetChildren()) do
        local ui = plot:FindFirstChild("UI")
        local bb = ui and ui:FindFirstChildOfClass("BillboardGui")

        if bb then
            local pn = bb:FindFirstChild("PlayerName", true)
            if pn and pn.Text and pn.Text:find(lp.Name, 1, true) then
                return plot
            end
        end
    end

    return nil
end

-- ===== Functions =====
local function doCollect()
    pcall(function()
        local plot = getMyPlot()
        if not plot then return end

        local plotFolder = plot:FindFirstChild("Plot")
        local buildings = plotFolder and plotFolder:FindFirstChild("Buildings")
        if not buildings then return end

        for _, building in ipairs(buildings:GetChildren()) do
            if building:IsA("Model") then
                pcall(function()
                    remote:FireServer({
                        building,
                        "H"
                    })
                end)
                task.wait(0.1)
            end
        end
    end)
end

local function doSell()
    pcall(function()
        local b = GetBridge("SellAll")
        if b then
            b:Fire()
        end
    end)
end

local function doBuy()
    for category, items in pairs(selectedItems) do
        for itemName, selected in pairs(items) do
            if selected then
                pcall(function()
                    remote:FireServer({
                        {
                            item = itemName,
                            shop = category
                        },
                        " "
                    })
                end)
                task.wait(0.15)
            end
        end
    end
end

local function doBuyBM()
    for itemName, selected in pairs(selectedBM) do
        if selected then
            pcall(function()
                remote:FireServer({
                    {
                        item = itemName,
                        shop = "BlackMarket"
                    },
                    "Z"
                })
            end)
            task.wait(0.15)
        end
    end
end

-- ===== Loop =====
task.spawn(function()
    while task.wait(CONFIG.interval) do
        if autoCollect then
            doCollect()
        end

        if autoSell then
            doSell()
        end

        if autoBuy then
            doBuy()
        end

        if autoBuyBM then
            doBuyBM()
        end
    end
end)

-- ===== UI =====
local Window = Fluent:CreateWindow({
    Title = "Mini War",
    SubTitle = "made by BatmanScript",
    TabWidth = 120,
    Size = UDim2.fromOffset(580, 460),
    Acrylic = true,
    Theme = "Dark",
    MinimizeKey = Enum.KeyCode.RightControl
})

local Tabs = {
    Main        = Window:AddTab({ Title = "Main", Icon = "home" }),
    Farm        = Window:AddTab({ Title = "Farm", Icon = "sprout" }),
    House       = Window:AddTab({ Title = "House", Icon = "house" }),
    Military    = Window:AddTab({ Title = "Military", Icon = "shield" }),
    Decor       = Window:AddTab({ Title = "Decor", Icon = "flower-2" }),
    BlackMarket = Window:AddTab({ Title = "Black Market", Icon = "store" }),
    Settings    = Window:AddTab({ Title = "Settings", Icon = "settings" }),
}

-- ===== Main Tab =====
Tabs.Main:AddSection("Collect")

Tabs.Main:AddToggle("AutoCollect", {
    Title = "Auto Collect",
    Default = CONFIG.autoCollect,
    Callback = function(v)
        autoCollect = v
        if v then
            task.spawn(doCollect)
        end
    end
})

Tabs.Main:AddSection("Sell")

Tabs.Main:AddToggle("AutoSell", {
    Title = "Auto Sell",
    Default = CONFIG.autoSell,
    Callback = function(v)
        autoSell = v
        if v then
            task.spawn(doSell)
        end
    end
})

Tabs.Main:AddSection("Buy")

Tabs.Main:AddToggle("AutoBuy", {
    Title = "Auto Buy",
    Default = CONFIG.autoBuy,
    Callback = function(v)
        autoBuy = v
        if v then
            task.spawn(doBuy)
        end
    end
})

Tabs.Main:AddToggle("AutoBuyBM_Main", {
    Title = "Auto Buy Black Market",
    Default = CONFIG.autoBuyBM,
    Callback = function(v)
        autoBuyBM = v
        if v then
            task.spawn(doBuyBM)
        end
    end
})

Tabs.Main:AddButton({
    Title = "Collect Now",
    Callback = function()
        task.spawn(doCollect)
    end
})

Tabs.Main:AddButton({
    Title = "Sell Now",
    Callback = function()
        task.spawn(doSell)
    end
})

Tabs.Main:AddButton({
    Title = "Buy Selected Now",
    Callback = function()
        task.spawn(doBuy)
    end
})

Tabs.Main:AddButton({
    Title = "Buy Black Market Now",
    Callback = function()
        task.spawn(doBuyBM)
    end
})

-- ===== Building Tabs =====
local categories = {"Farm", "House", "Military", "Decor"}

for _, cat in ipairs(categories) do
    local tab = Tabs[cat]
    tab:AddSection("Selection Items")

    local items = {}

    for name, cfg in pairs(BuildingsConfig) do
        if type(cfg) == "table" and cfg.Type == cat then
            table.insert(items, {
                name = name,
                display = cfg.DisplayName or name,
                price = cfg.Price or 0
            })
        end
    end

    table.sort(items, function(a, b)
        return a.price < b.price
    end)

    for _, item in ipairs(items) do
        if selectedItems[cat][item.name] == nil then
            selectedItems[cat][item.name] = false
        end

        tab:AddToggle("toggle_" .. cat .. "_" .. item.name, {
            Title = item.display,
            Description = "Price: " .. tostring(item.price),
            Default = selectedItems[cat][item.name] or false,
            Callback = function(v)
                selectedItems[cat][item.name] = v
            end
        })
    end
end

-- ===== Black Market Tab =====
Tabs.BlackMarket:AddSection("Auto Buy")

Tabs.BlackMarket:AddToggle("AutoBuyBM", {
    Title = "Auto Buy Black Market",
    Default = CONFIG.autoBuyBM,
    Callback = function(v)
        autoBuyBM = v
        if v then
            task.spawn(doBuyBM)
        end
    end
})

Tabs.BlackMarket:AddButton({
    Title = "Buy Selected Black Market Now",
    Callback = function()
        task.spawn(doBuyBM)
    end
})

Tabs.BlackMarket:AddSection("Selection Items")

local bmItems = {}

for name, cfg in pairs(BuildingsConfig) do
    if type(cfg) == "table" and cfg.Type == "BlackMarket" then
        table.insert(bmItems, {
            name = name,
            display = cfg.DisplayName or name
        })
    end
end

if #bmItems == 0 and ShopsConfig.BlackMarket then
    for _, item in ipairs(ShopsConfig.BlackMarket) do
        local itemName = item.name or item.Name or item.Item or item
        if type(itemName) == "string" then
            table.insert(bmItems, {
                name = itemName,
                display = itemName
            })
        end
    end
end

for name, selected in pairs(selectedBM) do
    local exists = false

    for _, item in ipairs(bmItems) do
        if item.name == name then
            exists = true
            break
        end
    end

    if not exists then
        table.insert(bmItems, {
            name = name,
            display = name
        })
    end
end

table.sort(bmItems, function(a, b)
    return a.name < b.name
end)

for _, item in ipairs(bmItems) do
    if selectedBM[item.name] == nil then
        selectedBM[item.name] = false
    end

    Tabs.BlackMarket:AddToggle("toggle_BM_" .. item.name, {
        Title = item.display,
        Default = selectedBM[item.name] or false,
        Callback = function(v)
            selectedBM[item.name] = v
        end
    })
end

-- ===== Settings =====
SaveManager:SetLibrary(Fluent)
InterfaceManager:SetLibrary(Fluent)

SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({})

InterfaceManager:SetFolder("MiniWarHub")
SaveManager:SetFolder("MiniWarHub/miniwar")

InterfaceManager:BuildInterfaceSection(Tabs.Settings)
SaveManager:BuildConfigSection(Tabs.Settings)

Window:SelectTab(1)

Fluent:Notify({
    Title = "BatmanScript Miniwar",
    Content = "Script loaded!",
    Duration = 5
})

SaveManager:LoadAutoloadConfig()
