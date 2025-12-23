local VERSION = "2.8.0"

-- Services
local HttpService = game:GetService("HttpService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local StarterGui = game:GetService("StarterGui")
local TextChatService = game:GetService("TextChatService")
local UserInputService = game:GetService("UserInputService")

-- Config
local letters = {"A","B","C","D","E","F","G","H","I","J","K","L","M","N","O","P","Q","R","S","T","U","V","W","X","Y","Z"}
local userCooldowns, blockedPlayers, whiteListedPlayers, submittedAnswer = {}, {}, {}, {}
local answeredCorrectly, currentQuestion, questionAnsweredBy = {}, nil, nil
local quizRunning, awaitingAnswer, quizCooldown = false, false, false
local questionPoints, answerOptionsSaid = 1, 0
local timeSinceLastMessage = tick()
local minMessageCooldown = 2.3
local whiteListEnabled = false
local antiAfkEnabled = false

local settings = {
    questionTimeout = 10,
    userCooldown = 5,
    automaticLeaderboards = true,
    signStatus = true,
    romanNumbers = true,
    repeatTagged = true,
    removeLeavingPlayersFromLB = true,
    autoplay = false,
}

local maxCharactersInMessage = 200
if game.PlaceId == 5118029260 then maxCharactersInMessage = 100 end

-- Utils
local function notify(title, text, configTable, duration)
    configTable = configTable or {}
    local bindFunc = Instance.new("BindableFunction")
    StarterGui:SetCore("SendNotification", {
        Title = title,
        Text = text,
        Callback = bindFunc,
        Button1 = configTable.Button1,
        Button2 = configTable.Button2,
        Duration = duration
    })
    if configTable.Callback then bindFunc.OnInvoke = configTable.Callback end
end

local function shuffle(tbl)
    local rng = Random.new()
    for i = #tbl, 2, -1 do
        local j = rng:NextInteger(1, i)
        tbl[i], tbl[j] = tbl[j], tbl[i]
    end
    return tbl
end

local function intToRoman(num)
    local numberMap = {
        {1000,'M'},{900,'CM'},{500,'D'},{400,'CD'},{100,'C'},
        {90,'XC'},{50,'L'},{40,'XL'},{10,'X'},{9,'IX'},{5,'V'},{4,'IV'},{1,'I'}
    }
    local roman = ""
    for _, v in ipairs(numberMap) do
        while num >= v[1] do
            roman = roman..v[2]
            num = num - v[1]
        end
    end
    return roman
end

local function Chat(msg)
    -- Prevent spam & rate-limit
    if tick() - timeSinceLastMessage < minMessageCooldown then
        task.wait(minMessageCooldown - (tick() - timeSinceLastMessage))
    end
    if TextChatService.TextChannels:FindFirstChild("RBXGeneral") then
        TextChatService.TextChannels.RBXGeneral:SendAsync(msg)
    end
    timeSinceLastMessage = tick()
end

local function CalculateReadTime(text)
    local t = #string.split(text," ")*0.4
    return math.max(t, minMessageCooldown)
end

local function EscapePattern(pattern)
    return string.gsub(pattern,"[%(%)%.%%%+%-%*%?%[%]%^%$]","%%%1")
end

-- Anti-AFK
local function EnableAntiAfk()
    if antiAfkEnabled then return end
    local VirtualUser = cloneref(game:GetService("VirtualUser"))
    LocalPlayer.Idled:Connect(function()
        VirtualUser:CaptureController()
        VirtualUser:ClickButton2(Vector2.new())
    end)
    antiAfkEnabled = true
    notify("Anti-AFK Enabled","You will not get kicked for idling")
end

-- Data
local DATA_FILENAME = "quizbot_data.json"
local data = {}
if isfile and isfile(DATA_FILENAME) then
    local success, decoded = pcall(function()
        return HttpService:JSONDecode(readfile(DATA_FILENAME))
    end)
    if success then data = decoded end
end

local function writeToDataFile(key,value)
    data[key] = value
    if writefile then
        writefile(DATA_FILENAME,HttpService:JSONEncode(data))
    end
end
local function getDataFileValue(key) return data[key] end

-- Points
local pointManager,userPoints = {},{}
local function getDisplayName(player)
    return Players:FindFirstChild(player.Name) and Players:FindFirstChild(player.Name).DisplayName or player.Name
end

function pointManager.NewAccount(player)
    userPoints[player.Name] = {DisplayName=getDisplayName(player), GlobalPoints=0, CurrentQuizPoints=0, Streak=0}
    return userPoints[player.Name]
end

function pointManager.AddPoints(player,points,type)
    points = tonumber(points) or 1
    type = type or "All"
    local acc = userPoints[player.Name] or pointManager.NewAccount(player)
    points = points + math.floor(acc.Streak/3)
    if type=="All" then
        acc.GlobalPoints += points
        if quizRunning then acc.CurrentQuizPoints += points end
    elseif acc[type] then
        acc[type] += points
    end
    acc.Streak += 1
end

function pointManager.ResetStreak(player)
    if userPoints[player.Name] then userPoints[player.Name].Streak=0 end
end

-- Questions
local question = {}
question.__index = question

function question.New(text,options,value,hint)
    return setmetatable({
        mainQuestion=text,
        answers=options,
        rightAnswer=letters[1],
        rightAnswerIndex=1,
        value=value or 1,
        hint=hint or ""
    },question)
end

function question:GetHint()
    return self.hint ~= "" and self.hint or "Hint: consider your options carefully."
end

function question:Ask()
    if not quizRunning then return false end
    answerOptionsSaid = 0
    local rightAns = self.answers[self.rightAnswerIndex]
    self.answers = shuffle(self.answers)
    self.rightAnswerIndex = table.find(self.answers,rightAns)
    self.rightAnswer = letters[self.rightAnswerIndex]
    questionAnsweredBy = nil
    currentQuestion = self
    questionPoints = self.value

    if self.value>1 then
        Chat("⭐ | "..self.value.."x points for this question")
        task.wait(2)
    end

    Chat("🎙️ | "..self.mainQuestion)
    task.wait(CalculateReadTime(self.mainQuestion))

    for i,v in ipairs(self.answers) do
        Chat(letters[i]..") "..v)
        answerOptionsSaid = i
        task.wait(CalculateReadTime(v))
    end
end

-- Categories
local categoryManager,categories = {},{categoryList={},numberOfDifficulties={}}
categoryManager.__index = categoryManager
local difficultyOrder={"","easy","medium","hard"}

function categoryManager.New(catName,difficulty)
    difficulty = string.lower(difficulty or "")
    categories.categoryList[catName] = categories.categoryList[catName] or {}
    categories.numberOfDifficulties[catName] = categories.numberOfDifficulties[catName] or 0
    categories.categoryList[catName][difficulty] = categories.categoryList[catName][difficulty] or {}
    categories.numberOfDifficulties[catName] = categories.numberOfDifficulties[catName] + 1
    local newCat = {}
    table.insert(categories.categoryList[catName][difficulty],newCat)
    return setmetatable(newCat,categoryManager)
end

function categoryManager:Add(text,options,value,custom)
    local q = question.New(text,options,value)
    table.insert(custom or self,q)
end

-- Chat Listener
local function startChatListening(msg,player)
    msg = string.upper(msg or "")
    if not currentQuestion or questionAnsweredBy or table.find(userCooldowns,player.Name)
       or table.find(blockedPlayers,player.Name) or table.find(submittedAnswer,player.Name)
       or (whiteListEnabled and not table.find(whiteListedPlayers,player.Name)) then return end

    local matchAnswer,minLen = nil,4
    if #currentQuestion.answers[currentQuestion.rightAnswerIndex]<minLen then minLen=#currentQuestion.answers[currentQuestion.rightAnswerIndex] end
    local escMsg = EscapePattern(msg)

    if #msg >= minLen then
        for _,v in ipairs(currentQuestion.answers) do
            local escAns = EscapePattern(v)
            if v:upper() == msg or string.match(v:upper(),escMsg) or string.match(msg,escAns:upper()) then
                if matchAnswer then return end
                matchAnswer=v
            end
        end
    end

    -- Single letter answer
    if not matchAnswer and #msg==1 then
        local idx = table.find(letters,msg)
        if idx and currentQuestion.answers[idx] then matchAnswer=currentQuestion.answers[idx] end
    end

    if matchAnswer then
        questionAnsweredBy=player
        pointManager.AddPoints(player,questionPoints)
        Chat("✅ "..player.DisplayName.." answered correctly! Answer: "..matchAnswer)
    end
end

-- Auto-enable Anti-AFK
EnableAntiAfk()