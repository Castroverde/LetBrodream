local VERSION = "2.4.6"

local letters = {"A","B","C","D","E","F","G","H","I","J","K","L","M","N","O","P","Q","R","S","T","U","V","W","X","Y","Z"}
local userCooldowns = {}
local currentQuestion
local questionAnsweredBy
local quizRunning = false
local HttpService = game:GetService("HttpService")
local players = game:GetService("Players")
local localPlayer = players.LocalPlayer
local blockedPlayers = {}
local whiteListedplayers = {}
local mode = "Quiz"
local answeredCorrectly = {}
local submittedAnswer = {}
local awaitingAnswer = false
local questionPoints = 1
local timeSinceLastMessage = tick()
local placeId = game.PlaceId
local replicatedStorage = game:GetService("ReplicatedStorage")
local StarterGui = game:GetService("StarterGui")
local textChatService = game:GetService("TextChatService")
local quizCooldown = false
local answerOptionsSaid = 0 -- how many answer options have been said (0 = none, 1 = a, 2 = b, etc.). Prevents users from spamming letters before they even know what the corresponding answer option is
local minMessageCooldown = 2.3 -- how much you need to wait to send another message to avoid ratelimit
local whiteListEnabled = false
local ContextActionService = game:GetService("ContextActionService")
local UserInputService = game:GetService("UserInputService")

local settings = {
    questionTimeout = 10,
    userCooldown = 5,
    sendLeaderBoardAfterQuestions = 0, -- only send quiz LB at end of quiz by default
    automaticLeaderboards = true,
    automaticCurrentQuizLeaderboard = false,
    automaticServerQuizLeaderboard = true,
    signStatus = true,
    romanNumbers = true,
    autoplay = false,
    repeatTagged = true,
    sendDetailedCategorylist = false, -- off by default since sending it this way tends to trigger the filter. Blame Roblox
    removeLeavingPlayersFromLB = true,
}

local numberMap = {
    {1000, 'M'},
    {900, 'CM'},
    {500, 'D'},
    {400, 'CD'},
    {100, 'C'},
    {90, 'XC'},
    {50, 'L'},
    {40, 'XL'},
    {10, 'X'},
    {9, 'IX'},
    {5, 'V'},
    {4, 'IV'},
    {1, 'I'}
}

function intToRoman(num)
    local roman = ""
    while num > 0 do
        for _, v in pairs(numberMap) do
            local romanChar = v[2]
            local int = v[1]
            while num >= int do
                roman = roman..romanChar
                num = num - int
            end
        end
    end
    return roman
end

local oldChat
if textChatService.ChatVersion == Enum.ChatVersion.TextChatService then
    oldChat = false
else
    oldChat = true
end

local function Chat(msg)
    if not oldChat then
        textChatService.TextChannels.RBXGeneral:SendAsync(msg)
    else
        replicatedStorage.DefaultChatSystemChatEvents.SayMessageRequest:FireServer(msg, "All")
    end
end

local function Shuffle(tbl) -- Table shuffle function by sleitnick
    local rng = Random.new()
    for i = #tbl, 2, -1 do
        local j = rng:NextInteger(1, i)
        tbl[i], tbl[j] = tbl[j], tbl[i]
    end
    return tbl
end

local function roundNumber(num, numDecimalPlaces)
    return tonumber(string.format("%." .. (numDecimalPlaces or 0) .. "f", num))
end

-- callback value of configTable needs to be a bindable function
local notifyBindable = Instance.new("BindableFunction")
local function notify(title, text, configTable, duration)
    configTable = configTable or {}
    StarterGui:SetCore("SendNotification", {
        Title = title,
        Text = text,
        Callback = notifyBindable,
        Button1 = configTable.Button1,
        Button2 = configTable.Button2,
        Duration = duration
    })
    if configTable.Callback then
        notifyBindable.OnInvoke = configTable.Callback
    end
end

local function setgenv(key, value)
    pcall(function()
        getgenv()[key] = value
    end)
end

if getgenv and getgenv().QUIZBOT_RUNNING then
    notify("Quizbot is already running", "You cannot run two instances of quizbot at once")
    return
end
setgenv("QUIZBOT_RUNNING", true)

local function copyLatestScript() -- copies latest loadstring version of the script to clipboard
    setclipboard('loadstring(game:HttpGet("https://raw.githubusercontent.com/Damian-11/quizbot/master/quizbot.luau"))()')
    notify("Script copied", "The latest version of quizbot has been copied to your clipboard")
end

local DATA_FILENAME = "quizbot_data.json"
type dataValueType = string | number | boolean
local data = {}
if isfile and isfile(DATA_FILENAME) then
    data = HttpService:JSONDecode(readfile(DATA_FILENAME))
end

local function writeToDataFile(key, value)
    if not writefile then
        notify("Can't write to file", "Your exploit does not support the writefile function. Your settings will not be saved.")
        return
    end
    data[key] = value
    writefile(DATA_FILENAME, HttpService:JSONEncode(data))
end
local function getDataFileValue(key)
    return data[key]
end

-- check if running latest version from GitHub
local LATEST_VERSION_URL = "https://raw.githubusercontent.com/Damian-11/quizbot/master/version_number.luau"
local success, LATEST_VERSION = pcall(function()
    return loadstring(game:HttpGet(LATEST_VERSION_URL))() -- this returns a string, such as "2.5.7"
end)

local outdated = success and LATEST_VERSION and VERSION ~= LATEST_VERSION
if outdated then
    if not getDataFileValue("disableVersionAlert") then -- true if user has pressed "Don't show again" before
        notify(
            "Outdated quizbot version",
            "You are running an outdated version of quizbot. Click the button below to copy the latest version of this script",
            {
                Callback = copyLatestScript,
                Button1 = "Copy latest version",
            },
            10
        )
    end
end

local function EscapePattern(pattern) -- escapes magic characters in pattern
    local escapePattern = "[%(%)%.%%%+%-%*%?%[%]%^%$]"
    return string.gsub(pattern, escapePattern, "%%%1")
end

local antiFilteringDone = false
local importantMessageSent = false -- if an important message that needs to be resent if filtered has been sent recently
local messageBeforeFilter = ""
local answeredByAltMessage = "" -- alt message specially for the correct answer text
local mainQuestionSent = false
local messageFiltered = false -- set to false once main question gets asked successfully without being filtered
local lastMessageTime = 0 -- time at which the last message was picked up by the antifilering function
function SendMessageWhenReady(message, important, altMessage) -- sends message so roblox won't rate limit it. if message is "important", script will send it again if it gets filtered/tagged first time. Altmessage is the message to send instead of the original if it gets tagged
    if not quizRunning then
        return
    end
    if not settings.repeatTagged then
        important = false
    end
    if important then
        importantMessageSent = true
        messageBeforeFilter = message
        answeredByAltMessage = altMessage
        messageFiltered = false
        antiFilteringDone = false
    end
    if tick() - timeSinceLastMessage >= minMessageCooldown then
        Chat(message)
        timeSinceLastMessage = tick()
    else
        task.wait(minMessageCooldown - (tick() - timeSinceLastMessage))
        if not quizRunning then
            return
        end
        Chat(message)
        timeSinceLastMessage = tick()
    end
    if important then
        lastMessageTime = tick()
        while (not antiFilteringDone or mainQuestionSent) and quizRunning do -- yields until the anti filter functions have done their job
            task.wait()
            if tick() - lastMessageTime > 4.5 then -- prevents the antifiltering functions yielding forever in case the message fails to send
                notify("Error sending quiz message", "Chat rate limit reached. Avoid sending messages in the chat while a quiz is running")
                messageFiltered = true -- the message not sending at all is equivalent to it being filtered
                break
            end
        end
    end
    importantMessageSent = false
end

-- Booth Game integration may break at any point. I won't bother fixing it every time they update the remote events
local boothGame = false
local signRemote
local changeSignTextRemote
if placeId == 8351248417 then
    signRemote = replicatedStorage:WaitForChild("Remotes"):WaitForChild("SettingsRem")
    changeSignTextRemote = replicatedStorage.SharedModules.TextInputPrompt.TextInputEvent
    if signRemote and changeSignTextRemote then
        boothGame = true
    end
end
local function UpdateSignText(text)
    if not boothGame or not settings.signStatus or not text then -- only works in "booth game"
        return
    end
    local sign = localPlayer.Character:FindFirstChild("Text Sign") or localPlayer.Backpack:FindFirstChild("Text Sign")
    if not sign then
        return
    end
    signRemote:FireServer("SignServer")
    changeSignTextRemote:FireServer(text)
end

local maxCharactersInMessage = 200
if placeId == 5118029260 then -- GRP cuts down messages at 100 characters
    maxCharactersInMessage = 100
end

local endMessage = "Quiz ended"
if localPlayer.UserId == 2005147350 then
    endMessage = "Quiz ended"
end

local function CalculateReadTime(text)
    local timeToWait = #string.split(text, " ") * 0.4
    if timeToWait < minMessageCooldown then
        timeToWait = minMessageCooldown
    end
    return timeToWait
end
-------

--- Question OOP ---
local question = {}
question.__index = question

function question.New(quesitonText, options, value)
    local newQuestion = {}
    newQuestion.mainQuestion = quesitonText
    newQuestion.answers = options
    newQuestion.rightAnswer = letters[1]
    newQuestion.rightAnswerIndex = 1
    value = value or 1
    newQuestion.value = value
    setmetatable(newQuestion, question)
    return newQuestion
end

function question:Ask()
    if not quizRunning then
        return false
    end
    answerOptionsSaid = 0
    local rightAnswerBeforeShuffle = self.answers[self.rightAnswerIndex]
    self.answers = Shuffle(self.answers)
    self.rightAnswerIndex = table.find(self.answers, rightAnswerBeforeShuffle)
    self.rightAnswer = letters[self.rightAnswerIndex]
    if self.value > 1 then
        SendMessageWhenReady("⭐ | "..self.value.."x points for question")
        task.wait(2)
    end
    questionAnsweredBy = nil
    UpdateSignText(self.mainQuestion)
    currentQuestion = self
    questionPoints = self.value
    mainQuestionSent = true
    SendMessageWhenReady("🎙️ | "..self.mainQuestion, true)
    if messageFiltered then
        task.wait(3)
        Chat("➡️ | Repeated filtering detected. Skipping to the next question...")
        return false
    end
    local waitTime = CalculateReadTime(self.mainQuestion)
    local timeWaited = 0
    while waitTime > timeWaited do -- check if someone answered every 0.1ms while waiting to send options
        if questionAnsweredBy or not quizRunning then
            return true
        end
        task.wait(0.1)
        timeWaited += 0.1
    end
    for i, v in ipairs(self.answers) do
        if questionAnsweredBy or not quizRunning then
            return true
        end
        if i ~= 1 then
            task.wait(CalculateReadTime(v))
        end
        if questionAnsweredBy or not quizRunning then
            return true
        end
        SendMessageWhenReady(letters[i]..")"..v, true) -- 1 = A) 2 = B) 3 = C) etc.
        answerOptionsSaid = i
    end
end

local function splitIntoMessages(itemTable, separator, clearFilter, waitTime) -- split table into multiple messages to prevent roblox cutting down the message
    local tempItemList = {}
    local messages = {}
    local currentLength = 0
    for _, item in itemTable do
        if currentLength + #item + (#separator * #tempItemList) + 6 >= maxCharactersInMessage then -- maxCharactersInMessage characters is the limit for chat messages in Roblox. For each item, we are adding a separator. +6 at end for " [x/x]" at the end of message
            local concatTable = table.concat(tempItemList, separator)
            table.insert(messages, concatTable)
            table.clear(tempItemList)
            table.insert(tempItemList, item)
            currentLength = #item
        else
            table.insert(tempItemList, item)
            currentLength = currentLength + #item
        end
    end
    table.insert(messages, table.concat(tempItemList, separator))
    for messageIndex, message in ipairs(messages) do
        if quizRunning then
            return
        end
        local messageNumberString = string.format("(%d/%d)", messageIndex, #messages) -- [current message/amount of messages]
        if messageIndex == 2 or messageIndex == 3 then
            if clearFilter then
                Chat("Waiting for filter...") -- hacky solution that prevents the second message getting filtered (sometimes works)
            end
            task.wait(6)
        end
        if quizRunning then
            return
        end
        Chat(message.." "..messageNumberString)
        task.wait(waitTime or CalculateReadTime(message) * 0.7) -- multiplied by 0.7 because full read time is too long for longer texts
    end
end

local antiAfkEnabled = false
local function EnableAntiAfk() -- prevents Roblox kicking the player after 20 minutes of inactivity
    -- credit to IY for antiafk code: https://github.com/EdgeIY/infiniteyield/tree/master
    if antiAfkEnabled then
        return
    end
    local GC = getconnections or get_signal_cons
    if GC then
        for i, v in pairs(GC(localPlayer.Idled)) do
            if v["Disable"] then
                v["Disable"](v)
            elseif v["Disconnect"] then
                v["Disconnect"](v)
            end
        end
    else
        local VirtualUser = game:GetService("VirtualUser")
        localPlayer.Idled:Connect(function()
            VirtualUser:CaptureController()
            VirtualUser:ClickButton2(Vector2.new())
        end)
    end
    antiAfkEnabled = true
    notify("Anti-AFK enabled successfully", "You will not get kicked for being idle")
end

--- Category OOP ---
local categoryManager = {}
local categories = {}
categories.categoryList = {}
categories.numberOfDifficulties = {}
categoryManager.__index = categoryManager

--[[Category table reference:
categories = {
    categoryList = {
        [categoryName] = {
            easy = {{quiz1Questions}, {quiz2Questions}},
            medium = {...},
            ...
        },
        [categoryName2] = {
            easy = {{quiz1Questions}, {quiz2Questions}},
            medium = {...},
            ...
        }
    }
    numberOfDifficulties = {
        [categoryName] = 2
        [categoryName2] = 3
        ...
    }
}
]]--

local difficultyOrder = {"", "easy", "medium", "hard"} -- the order in which difficulties should appear when sending the category list
function categoryManager.New(categoryName, difficulty)
    difficulty = difficulty or "" -- if difficulty is not specified, use blank
    difficulty = string.lower(difficulty)
    if not categories.categoryList[categoryName] then
        categories.categoryList[categoryName] = {}
        categories.numberOfDifficulties[categoryName] = 0
    end
    if not categories.categoryList[categoryName][difficulty] then
        categories.categoryList[categoryName][difficulty] = {}
        categories.numberOfDifficulties[categoryName] += 1
        if not table.find(difficultyOrder, difficulty) then
            table.insert(difficultyOrder, difficulty)
        end
    end
    table.insert(categories.categoryList[categoryName][difficulty], {})
    local newCategory = categories.categoryList[categoryName][difficulty][#categories.categoryList[categoryName][difficulty]] -- get the new category at the end of its difficulty table
    setmetatable(newCategory, categoryManager)
    return newCategory
end

function categoryManager:Add(quesitonText, options, value, customQuestion)
    self = customQuestion or self
    local newQuestion = question.New(quesitonText, options, value)
    table.insert(self, newQuestion)
end

-- Adding existing categories
local mathematicsHard = categoryManager.New("Mathematics", "hard")
mathematicsHard:Add("What is the value of Pi to 5 decimal places?", {"3.14159", "3.14158", "3.14157", "3.14156"})
mathematicsHard:Add("What is the derivative of sin(x)?", {"cos(x)", "-cos(x)", "sin(x)", "-sin(x)"})
mathematicsHard:Add("What is the integral of 1/x?", {"ln|x| + C", "e^x + C", "x^2/2 + C", "1/x + C"})
mathematicsHard:Add("What is the solution to the equation 2x^2 - 3x + 1 = 0?", {"x = 1 or x = 0.5", "x = 1.5 or x = -0.5", "x = 2 or x = -1", "x = 0.5 or x = -0.5"})
mathematicsHard:Add("What is the sum of the infinite geometric series 1 + 1/2 + 1/4 + 1/8 + ... ?", {"2", "1", "2.5",
