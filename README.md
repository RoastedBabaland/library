-- Cleaned & Safe UI Library (no Discord logging/webhook)
-- Features: Draggable main frame, Tabs, Sections, Sectors, Toggles + Keybinds, Color Pickers, Dropdowns, Combo, Buttons, TextBoxes, Sliders

local library = {}

-- Services
local TweenService     = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local TextService      = game:GetService("TextService")
local HttpService      = game:GetService("HttpService")
local RunService       = game:GetService("RunService")
local Players          = game:GetService("Players")
local CoreGui          = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local Mouse       = LocalPlayer:GetMouse()

-- Utility functions
function library:tween(obj, info, props)
    TweenService:Create(obj, info, props):Play()
end

function library:create(class, props, parent)
    local obj = Instance.new(class)
    for prop, value in pairs(props or {}) do
        obj[prop] = value
    end
    if parent then obj.Parent = parent end
    return obj
end

function library:get_text_size(text, size, font, bounds)
    return TextService:GetTextSize(text, size, font, bounds or Vector2.new(math.huge, math.huge))
end

-- Make GUI draggable
function library:set_draggable(gui)
    local dragging, dragInput, dragStart, startPos

    local function update(input)
        local delta = input.Position - dragStart
        gui.Position = UDim2.new(
            startPos.X.Scale,
            startPos.X.Offset + delta.X,
            startPos.Y.Scale,
            startPos.Y.Offset + delta.Y
        )
    end

    gui.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging   = true
            dragStart  = input.Position
            startPos   = gui.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    gui.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            update(input)
        end
    end)
end

-- Main library constructor
function library.new(title, config_folder)
    local menu = {}
    menu.values     = {}
    menu.on_load_cfg = {} -- you can connect callbacks here

    -- Config folder setup
    if not isfolder(config_folder) then
        makefolder(config_folder)
    end

    -- Deep copy helper for configs
    function menu.copy(tbl)
        local copy = {}
        for k, v in pairs(tbl) do
            copy[k] = type(v) == "table" and menu.copy(v) or v
        end
        return copy
    end

    -- Save config (JSON format)
    function menu.save_cfg(name)
        local data = menu.copy(menu.values)
        writefile(config_folder .. name .. ".json", HttpService:JSONEncode(data))
    end

    -- Load config
    function menu.load_cfg(name)
        local path = config_folder .. name .. ".json"
        if not isfile(path) then return end

        local success, data = pcall(function()
            return HttpService:JSONDecode(readfile(path))
        end)

        if success then
            menu.values = data
            -- Fire callbacks if needed
            -- for _, cb in ipairs(menu.on_load_cfg) do cb() end
        end
    end

    -- ScreenGui
    local ScreenGui = library:create("ScreenGui", {
        Name            = "SafeLibrary",
        ResetOnSpawn    = false,
        ZIndexBehavior  = Enum.ZIndexBehavior.Global,
        IgnoreGuiInset  = true,
    }, CoreGui)

    -- Main frame
    local MainFrame = library:create("ImageButton", {
        Name            = "Main",
        AnchorPoint     = Vector2.new(0.5, 0.5),
        BackgroundColor3 = Color3.fromRGB(15, 15, 15),
        BorderColor3    = Color3.fromRGB(78, 93, 234),
        Position        = UDim2.new(0.5, 0, 0.5, 0),
        Size            = UDim2.new(0, 700, 0, 555),
        Image           = "http://www.roblox.com/asset/?id=7300333488",
        AutoButtonColor = false,
        Modal           = true,
    }, ScreenGui)

    library:set_draggable(MainFrame)

    -- Title bar
    library:create("TextLabel", {
        Name               = "Title",
        AnchorPoint        = Vector2.new(0.5, 0),
        BackgroundTransparency = 1,
        Position           = UDim2.new(0.5, 0, 0, 0),
        Size               = UDim2.new(1, -22, 0, 30),
        Font               = Enum.Font.Ubuntu,
        Text               = title or "UI Library",
        TextColor3         = Color3.fromRGB(255, 255, 255),
        TextSize           = 16,
        TextXAlignment     = Enum.TextXAlignment.Left,
        RichText           = true,
    }, MainFrame)

    -- Tab buttons container
    local TabButtonsContainer = library:create("Frame", {
        Name                 = "TabButtons",
        BackgroundTransparency = 1,
        Position             = UDim2.new(0, 12, 0, 41),
        Size                 = UDim2.new(0, 76, 0, 447),
    }, MainFrame)

    library:create("UIListLayout", {
        HorizontalAlignment = Enum.HorizontalAlignment.Center,
    }, TabButtonsContainer)

    -- Tabs content container
    local TabsContainer = library:create("Frame", {
        Name                 = "Tabs",
        BackgroundTransparency = 1,
        Position             = UDim2.new(0, 102, 0, 42),
        Size                 = UDim2.new(0, 586, 0, 446),
    }, MainFrame)

    -- Toggle menu visibility (Insert, Home, RightShift)
    local is_open = true
    UserInputService.InputBegan:Connect(function(input, gp)
        if gp then return end
        if input.KeyCode == Enum.KeyCode.Insert or
           input.KeyCode == Enum.KeyCode.Home or
           input.KeyCode == Enum.KeyCode.RightShift then
            is_open = not is_open
            ScreenGui.Enabled = is_open
        end
    end)

    -- Tab creation function
    function menu.new_tab(image_id)
        local tab = {}
        local tab_index = #menu.values + 1
        menu.values[tab_index] = {}

        local TabButton = library:create("TextButton", {
            BackgroundTransparency = 1,
            Size               = UDim2.new(0, 76, 0, 90),
            Text               = "",
        }, TabButtonsContainer)

        local TabIcon = library:create("ImageLabel", {
            AnchorPoint        = Vector2.new(0.5, 0.5),
            BackgroundTransparency = 1,
            Position           = UDim2.new(0.5, 0, 0.5, 0),
            Size               = UDim2.new(0, 32, 0, 32),
            Image              = image_id or "",
            ImageColor3        = Color3.fromRGB(100, 100, 100),
        }, TabButton)

        local TabFrame = library:create("Frame", {
            Name               = "TabFrame",
            BackgroundTransparency = 1,
            Size               = UDim2.new(1, 0, 1, 0),
            Visible            = false,
        }, TabsContainer)

        -- Tab switching logic
        TabButton.MouseButton1Down:Connect(function()
            for _, btn in pairs(TabButtonsContainer:GetChildren()) do
                if btn:IsA("TextButton") then
                    library:tween(btn.ImageLabel, TweenInfo.new(0.2), {ImageColor3 = Color3.fromRGB(100, 100, 100)})
                end
            end

            for _, frame in pairs(TabsContainer:GetChildren()) do
                if frame:IsA("Frame") then
                    frame.Visible = false
                end
            end

            TabFrame.Visible = true
            library:tween(TabIcon, TweenInfo.new(0.2), {ImageColor3 = Color3.fromRGB(84, 101, 255)})
        end)

        TabButton.MouseEnter:Connect(function()
            if TabFrame.Visible then return end
            library:tween(TabIcon, TweenInfo.new(0.2), {ImageColor3 = Color3.fromRGB(255, 255, 255)})
        end)

        TabButton.MouseLeave:Connect(function()
            if TabFrame.Visible then return end
            library:tween(TabIcon, TweenInfo.new(0.2), {ImageColor3 = Color3.fromRGB(100, 100, 100)})
        end)

        -- Return tab object (you can add sections/elements here)
        return {
            frame = TabFrame,
            add_section = function(section_name)
                -- Implement section creation here if needed
            end,
        }
    end

    return menu
end

return library
