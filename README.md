-- ============================================================  
-- Azure Nova UI  
-- DX9-based menu with working tabs, toggles, and  
-- RGB sliders for custom themes.  
-- Added Notifications System  
-- Integrated Box ESP  
-- v2.0.48-ui | Fix: scan Players service, never self-target  
-- ============================================================  
  
local CFG = {  
    VERSION = "2.0.48-ui",  
    SCRIPT_NAME = "Azure Nova",  
    DEFAULT_MENU_KEY = "Insert",  
    MENU_KEY_LABEL = "Insert",  
    TABS = { "Main Menu", "Settings", "Credits" },  
}  
  
if not _G.AzureNovaUI then _G.AzureNovaUI = {} end  
local S = _G.AzureNovaUI  
  
if S.stateVersion ~= CFG.VERSION then  
    S.activeTab       = 1  
    S.menuKey         = "Insert"  
    S.fpsToggle       = false  
    S.watermarkToggle = true  
    S.espMaster       = false  
    S.espTeams        = false  
    S.prevKeySig      = "none"  
    S.prevMouseDown   = false  
    S.uiColorR        = 0  
    S.uiColorG        = 170  
    S.uiColorB        = 255  
    S.draggingSliderR      = false  
    S.draggingSliderG      = false  
    S.draggingSliderB      = false  
    S.draggingSliderSmooth = false  
    S.draggingSliderDist   = false  
    S.draggingSliderSens   = false  
    S.menuX       = 96  
    S.menuY       = 92  
    S.draggingUI  = false  
    S.dragOffsetX = 0  
    S.dragOffsetY = 0  
    S.notifications  = {}  
    S.espMaxDistance = 10000  
    S.universalOpen  = true  
    S.lockOn       = false  
    S.lockOnDist   = 500  
    S.lockOnSmooth = 1  
    S.lockOnSens   = 100  
    S.lockOnFov    = 150  
    S.guiOn        = true  
    S.initialized  = false  
  
    S.cachedPath   = nil  
    S.pathStatus   = "Path Not Found"  
    S.pathColor    = {255, 80, 80}  
    S.rescanning   = false  
  
    S.stateVersion = CFG.VERSION  
end  
  
S.guiOn           = (S.guiOn ~= false)  
S.initialized     = (S.initialized == true)  
S.menuKey         = S.menuKey or "Insert"  
S.fpsToggle       = (S.fpsToggle == true)  
S.watermarkToggle = (S.watermarkToggle ~= false)  
S.espMaster       = (S.espMaster == true)  
S.espTeams        = (S.espTeams == true)  
S.prevKeySig      = S.prevKeySig or "none"  
S.prevMouseDown   = (S.prevMouseDown == true)  
S.uiColorR        = tonumber(S.uiColorR) or 0  
S.uiColorG        = tonumber(S.uiColorG) or 170  
S.uiColorB        = tonumber(S.uiColorB) or 255  
S.draggingSliderR      = (S.draggingSliderR == true)  
S.draggingSliderG      = (S.draggingSliderG == true)  
S.draggingSliderB      = (S.draggingSliderB == true)  
S.draggingSliderSmooth = (S.draggingSliderSmooth == true)  
S.draggingSliderDist   = (S.draggingSliderDist == true)  
S.draggingSliderSens   = (S.draggingSliderSens == true)  
S.menuX       = tonumber(S.menuX) or 96  
S.menuY       = tonumber(S.menuY) or 92  
S.draggingUI  = (S.draggingUI == true)  
S.dragOffsetX = tonumber(S.dragOffsetX) or 0  
S.dragOffsetY = tonumber(S.dragOffsetY) or 0  
S.notifications  = S.notifications or {}  
S.espMaxDistance = tonumber(S.espMaxDistance) or 10000  
S.universalOpen  = (S.universalOpen ~= false)  
S.activeTab      = tonumber(S.activeTab) or 1  
S.lockOn         = (S.lockOn == true)  
S.lockOnDist     = tonumber(S.lockOnDist) or 500  
S.lockOnSmooth   = tonumber(S.lockOnSmooth) or 1  
S.lockOnSens     = tonumber(S.lockOnSens) or 100  
S.lockOnFov      = tonumber(S.lockOnFov) or 150  
S.cachedPath     = S.cachedPath  
S.pathStatus     = S.pathStatus or "Path Not Found"  
S.pathColor      = S.pathColor or {255, 80, 80}  
S.rescanning     = (S.rescanning == true)  
  
if dx9.ShowConsole then dx9.ShowConsole(true) end  
  
-- ============================================================  
-- FPS TRACKER  
-- ============================================================  
if not _G.AzureNovaFPS then  
    _G.AzureNovaFPS = { LastTick=os.clock(), Frames=0, FPS=0 }  
end  
_G.AzureNovaFPS.Frames = _G.AzureNovaFPS.Frames + 1  
if os.clock() - _G.AzureNovaFPS.LastTick >= 1 then  
    _G.AzureNovaFPS.FPS      = _G.AzureNovaFPS.Frames  
    _G.AzureNovaFPS.Frames   = 0  
    _G.AzureNovaFPS.LastTick = os.clock()  
end  
  
local function getUserName()  
    if _G.AzureNovaUser and _G.AzureNovaUser ~= "User" then return _G.AzureNovaUser end  
    local n = "User"  
    pcall(function()  
        local v = os.getenv("USERNAME") or os.getenv("USER")  
        if v and v ~= "" then n = v end  
    end)  
    if n ~= "User" then _G.AzureNovaUser = n end  
    return n  
end  
  
-- ============================================================  
-- NOTIFICATIONS  
-- ============================================================  
local function addNotification(text, duration)  
    table.insert(S.notifications, 1, { text=text, duration=duration or 3, startTime=os.clock() })  
end  
  
-- ============================================================  
-- UTILITIES  
-- ============================================================  
local VK       = { insert=0x2D }  
local VK_NAMES = { [0x2D]="Insert" }  
  
local function clamp(v,lo,hi) return v<lo and lo or (v>hi and hi or v) end  
  
local function normalizeKeyName(key)  
    return tostring(key or ""):lower():gsub("%[",""):gsub("%]",""):gsub("[%s_%-%+]+","")  
end  
local function getKeySig(key)  
    if key==nil or key==0 or key=="" then return "none" end  
    if type(key)=="number" then return "n:"..tostring(key) end  
    local n=normalizeKeyName(key)  
    return n=="" and "none" or ("s:"..n)  
end  
local function keyMatches(key,target)  
    local a=normalizeKeyName(key); local b=normalizeKeyName(target)  
    if a=="" or b=="" then return false end  
    if type(key)=="number" then return key==(VK[b] or VK[a] or -1) end  
    return a==b or a==("vk"..b) or ("vk"..a)==b  
end  
local function keyToText(key)  
    if key==nil or key=="" then return "Unknown" end  
    if type(key)=="number" then return VK_NAMES[key] or tostring(key) end  
    local n=normalizeKeyName(key)  
    if n=="vkinsert" then return "Insert" end  
    return tostring(key):gsub("^%[",""):gsub("%]$","")  
end  
local function getMouse()  
    local m=dx9.GetMouse()  
    if type(m)=="table" then  
        return tonumber(m.x or m[1]) or 0, tonumber(m.y or m[2]) or 0  
    end  
    return 0,0  
end  
local function pointInRect(px,py,x,y,w,h)  
    return px>=x and px<=x+w and py>=y and py<=y+h  
end  
local function themeFor(r,g,b)  
    r=clamp(tonumber(r) or 0,  0,255)  
    g=clamp(tonumber(g) or 170,0,255)  
    b=clamp(tonumber(b) or 255,0,255)  
    return {  
        BG={0,0,0}, PANEL={12,24,46}, PANEL_2={16,34,62},  
        BORDER={clamp(r-24,0,255),clamp(g-34,0,255),clamp(b-14,0,255)},  
        ACCENT={r,g,b}, TEXT={234,242,255}, DIM={156,180,208},  
    }  
end  
  
-- ============================================================  
-- DRAW HELPERS  
-- ============================================================  
local function drawBox(x,y,w,h,fill,border)  
    dx9.DrawFilledBox({x,y},{x+w,y+h},fill)  
    dx9.DrawBox({x,y},{x+w,y+h},border)  
end  
local function drawLine(x1,y1,x2,y2,col) dx9.DrawLine({x1,y1},{x2,y2},col) end  
local function drawText(x,y,col,text)    dx9.DrawString({x,y},col,text) end  
local function drawTab(x,y,w,h,text,active,T)  
    drawBox(x,y,w,h,active and T.PANEL_2 or T.PANEL,active and T.ACCENT or T.BORDER)  
    drawText(x+8,y+4,active and T.TEXT or T.DIM,text)  
    if active then dx9.DrawFilledBox({x,y+h-2},{x+w,y+h},T.ACCENT) end  
end  
local function drawToggle(x,y,w,text,value,T)  
    drawBox(x,y,w,24,T.PANEL,T.BORDER)  
    drawText(x+10,y+4,value and T.TEXT or T.DIM,text)  
    local tX=x+w-44; local tY=y+5  
    drawBox(tX,tY,34,14,value and T.ACCENT or {30,30,30},T.BORDER)  
    local thX=value and (tX+22) or (tX+2)  
    dx9.DrawFilledBox({thX,tY+2},{thX+10,tY+12},value and {255,255,255} or T.DIM)  
end  
local function drawSlider(x,y,w,title,value,maxVal,T,displayFn)  
    local maxV=tonumber(maxVal) or 100  
    local val=clamp(tonumber(value) or 0,0,maxV)  
    local fillW=math.max(1,math.floor(w*val/maxV))  
    local disp=displayFn and displayFn(val) or tostring(math.floor(val))  
    drawText(x,y,T.TEXT,title..": "..disp)  
    dx9.DrawFilledBox({x,y+16},{x+w,y+22},T.BORDER)  
    dx9.DrawFilledBox({x,y+16},{x+fillW,y+22},T.ACCENT)  
    dx9.DrawFilledBox({x+fillW-2,y+14},{x+fillW+2,y+24},T.TEXT)  
end  
  
-- ============================================================  
-- SAFE WRAPPERS  
-- ============================================================  
local function ffc(inst,name)  
    if not inst or inst==0 then return nil end  
    local ok,r=pcall(dx9.FindFirstChild,inst,name)  
    return (ok and r and r~=0) and r or nil  
end  
local function getChildrenSafe(obj)  
    if not obj or obj==0 then return {} end  
    local ok,c=pcall(dx9.GetChildren,obj)  
    return (ok and c) or {}  
end  
local function getPos(obj)  
    if not obj or obj==0 then return nil end  
    local ok,p=pcall(dx9.GetPosition,obj)  
    return (ok and p and p.x) and p or nil  
end  
local function w2s(pos)  
    if not pos then return nil end  
    local ok,sp=pcall(dx9.WorldToScreen,{pos.x,pos.y,pos.z})  
    return (ok and sp and sp.x) and sp or nil  
end  
local function dist3(a,b)  
    if not a or not b then return math.huge end  
    local dx=a.x-b.x; local dy=a.y-b.y; local dz=a.z-b.z  
    return math.sqrt(dx*dx+dy*dy+dz*dz)  
end  
  
local function getLocalCharacter(localPlayer)  
    if not localPlayer then return nil end  
    local ok,char=pcall(dx9.GetCharacter,localPlayer)  
    if ok and char and char~=0 then  
        return char  
    end  
    return nil  
end  
  
local function runRescan()  
    S.rescanning = true  
    S.cachedPath = nil  
    S.pathStatus = "Path Not Found"  
    S.pathColor = {255, 80, 80}  
  
    local ok, game = pcall(dx9.GetDatamodel)  
    if not ok or not game then  
        S.rescanning = false  
        return  
    end  
  
    local workspace = ffc(game, "Workspace")  
    if not workspace then  
        S.rescanning = false  
        return  
    end  
  
    local function hasTargetPath(m)  
        return ffc(m,"Torso") or ffc(m,"UpperTorso") or ffc(m,"HumanoidRootPart")  
    end  
  
    local function scanNode(node, currentPath)  
        if not node or node == 0 or S.cachedPath then return end  
  
        if hasTargetPath(node) then  
            S.cachedPath = currentPath  
            S.pathStatus = "Found path"  
            S.pathColor = {80, 255, 80}  
            return  
        end  
  
        local children = getChildrenSafe(node)  
        for _, child in next, children do  
            if S.cachedPath then return end  
            scanNode(child, currentPath .. "/" .. tostring(child))  
        end  
    end  
  
    scanNode(workspace, "Workspace")  
    S.rescanning = false  
end  
  
-- ============================================================  
-- GET OTHER PLAYERS' CHARACTERS  
-- ============================================================  
local function getEnemyCharacters(game, localPlayer)  
    local chars = {}  
    local ps = ffc(game, "Players")  
    if not ps then return chars end  
  
    local myChar = getLocalCharacter(localPlayer)  
    local players = getChildrenSafe(ps)  
  
    for _, player in next, players do  
        local ok, char = pcall(dx9.GetCharacter, player)  
        if ok and char and char ~= 0 and char ~= myChar then  
            table.insert(chars, char)  
        end  
    end  
  
    return chars  
end  
  
-- ============================================================  
-- ESP  
-- ============================================================  
local function collectEntities(workspace, localChar)  
    local results={}  
  
    local function hasTorso(m)  
        return ffc(m,"Torso") or ffc(m,"UpperTorso") or ffc(m,"HumanoidRootPart")  
    end  
  
    for _,child in next,getChildrenSafe(workspace) do  
        if child ~= localChar and hasTorso(child) then  
            table.insert(results,child)  
        else  
            for _,sub in next,getChildrenSafe(child) do  
                if sub ~= localChar and hasTorso(sub) then  
                    table.insert(results,sub)  
                end  
            end  
        end  
    end  
    return results  
end  
  
local function buildCharacterTeamMap()  
    local game=dx9.GetDatamodel(); if not game then return {} end  
    local ps=ffc(game,"Players"); if not ps then return {} end  
    local players=getChildrenSafe(ps)  
    local nameToIdx,count={},0  
    for _,p in next,players do  
        local ok,t=pcall(dx9.GetTeam,p)  
        if ok and t and type(t)=="string" and t~="" then  
            if not nameToIdx[t] then count=count+1; nameToIdx[t]=count end  
        end  
    end  
    local map={}  
    for _,p in next,players do  
        local ok1,t=pcall(dx9.GetTeam,p)  
        if ok1 and t and nameToIdx[t] then  
            local ok2,c=pcall(dx9.GetCharacter,p)  
            if ok2 and c then map[c]=nameToIdx[t] end  
        end  
    end  
    return map  
end  
  
local function drawESP()  
    if not S.espMaster then return end  
    local ok,game=pcall(dx9.GetDatamodel); if not ok or not game then return end  
    local lp=dx9.get_localplayer(); if not lp then return end  
  
    local myChar = getLocalCharacter(lp)  
    local myPos = lp.Position  
    if myChar then  
        local hrp = ffc(myChar, "HumanoidRootPart")  
        if hrp then  
            local hp = getPos(hrp)  
            if hp then myPos = hp end  
        end  
    end  
    if not myPos then return end  
  
    local T=themeFor(S.uiColorR,S.uiColorG,S.uiColorB)  
    local workspace=ffc(game,"Workspace"); if not workspace then return end  
  
    local entities=collectEntities(workspace, myChar)  
    local teamMap=S.espTeams and buildCharacterTeamMap() or {}  
    local halfH={Torso=1,UpperTorso=0.8}  
  
    for _,entity in next,entities do  
        local torso,tName=ffc(entity,"Torso"),"Torso"  
        if not torso then torso=ffc(entity,"UpperTorso"); tName="UpperTorso" end  
        if not torso then torso=ffc(entity,"HumanoidRootPart") end  
        if torso then  
            local pos=getPos(torso)  
            if pos and dist3(pos,myPos)<=S.espMaxDistance then  
                local hh=halfH[tName] or 1  
                local top=w2s({x=pos.x,y=pos.y+hh,z=pos.z})  
                local bot=w2s({x=pos.x,y=pos.y-hh,z=pos.z})  
                if top and bot then  
                    local sh=math.max(2,math.abs(top.y-bot.y))  
                    local bx=(top.x+bot.x)*0.5  
                    local ty=math.min(top.y,bot.y); local by=math.max(top.y,bot.y)  
                    local col=T.ACCENT  
                    if S.espTeams then  
                        local idx=teamMap[entity]  
                        if idx==1 then col={50,255,50} elseif idx==2 then col={255,50,50} end  
                    end  
                    dx9.DrawBox({bx-sh/2,ty},{bx+sh/2,by},col)  
                end  
            end  
        end  
    end  
end  
  
-- ============================================================  
-- MENU  
-- ============================================================  
local function renderMenu(mx,my,mouseDown,mousePressed)  
    local T=themeFor(S.uiColorR,S.uiColorG,S.uiColorB)  
    local gx,gy,gw,gh=S.menuX,S.menuY,410,420  
  
    if mousePressed and pointInRect(mx,my,gx,gy,gw,26) then  
        S.draggingUI=true; S.dragOffsetX=mx-gx; S.dragOffsetY=my-gy  
    end  
    if S.activeTab<1 or S.activeTab>#CFG.TABS then S.activeTab=1 end  
  
    drawBox(gx,gy,gw,gh,T.BG,T.BORDER)  
    drawBox(gx,gy,gw,26,T.PANEL,T.BORDER)  
    dx9.DrawFilledBox({gx,gy+24},{gx+gw,gy+26},T.ACCENT)  
    drawText(gx+10,gy+6,T.TEXT,CFG.SCRIPT_NAME.." v"..CFG.VERSION)  
  
    local tabX,tabY,tabW,tabH=gx+10,gy+34,120,22  
    for i=1,#CFG.TABS do  
        drawTab(tabX,tabY,tabW,tabH,CFG.TABS[i],S.activeTab==i,T)  
        if mousePressed and pointInRect(mx,my,tabX,tabY,tabW,tabH) then S.activeTab=i end  
        tabX=tabX+tabW+8  
    end  
  
    local cx=gx+10; local cy=gy+66; local cw=gw-20; local ch=gh-76  
    dx9.DrawBox({cx,cy},{cx+cw,cy+ch},T.BORDER)  
    local function lbl(t,col,x,y) drawText(x or cx+12,y,col or T.TEXT,t) end  
  
    if S.activeTab==1 then  
        lbl("Welcome, "..getUserName(),T.ACCENT,cx+12,cy+12)  
        local divY=cy+32  
        drawLine(cx+12,divY,cx+cw-12,divY,T.BORDER)  
        local hY=divY+4; local hH=20; local rowW=cw-24  
        lbl((S.universalOpen and "v" or ">").."  Universal",T.DIM,cx+12,hY+2)  
        if mousePressed and pointInRect(mx,my,cx+12,hY,rowW,hH) then  
            S.universalOpen=not S.universalOpen  
        end  
        if S.universalOpen then  
            local dbY=hY+hH+4  
            drawToggle(cx+12,dbY,   rowW,"ESP Players",S.espMaster,T)  
            drawToggle(cx+12,dbY+30,rowW,"ESP Teams",  S.espTeams, T)  
            drawToggle(cx+12,dbY+60,rowW,"Lock-on",    S.lockOn,   T)  
            local sSmY=dbY+94; local sDistY=sSmY+34; local sSensY=sDistY+34  
            drawSlider(cx+12,sSmY,  rowW,"Smooth (1=hard)", S.lockOnSmooth,10,  T)  
            drawSlider(cx+12,sDistY,rowW,"Lock Dist (studs)",S.lockOnDist, 1000,T)  
            drawSlider(cx+12,sSensY,rowW,"Sens Divisor",    S.lockOnSens,  200, T,  
                function(v) return string.format("%.2f",v/100) end)  
            if mousePressed then  
                if     pointInRect(mx,my,cx+12,dbY,   rowW,24) then  
                    S.espMaster=not S.espMaster  
                    addNotification("ESP "..(S.espMaster and "Enabled" or "Disabled"),2)  
                elseif pointInRect(mx,my,cx+12,dbY+30,rowW,24) then  
                    S.espTeams=not S.espTeams  
                    addNotification("ESP Teams "..(S.espTeams and "Enabled" or "Disabled"),2)  
                elseif pointInRect(mx,my,cx+12,dbY+60,rowW,24) then  
                    S.lockOn=not S.lockOn  
                    addNotification("Lock-on "..(S.lockOn and "Enabled" or "Disabled"),2)  
                end  
                if pointInRect(mx,my,cx+12,sSmY+16, rowW,6) then S.draggingSliderSmooth=true end  
                if pointInRect(mx,my,cx+12,sDistY+16,rowW,6) then S.draggingSliderDist=true  end  
                if pointInRect(mx,my,cx+12,sSensY+16,rowW,6) then S.draggingSliderSens=true  end  
            end  
            if mouseDown then  
                local rel=clamp(mx-(cx+12),0,rowW)  
                if     S.draggingSliderSmooth then S.lockOnSmooth=clamp(math.floor(rel/rowW*10  +0.5),1,  10)  
                elseif S.draggingSliderDist   then S.lockOnDist  =clamp(math.floor(rel/rowW*1000+0.5),10,1000)  
                elseif S.draggingSliderSens   then S.lockOnSens  =clamp(math.floor(rel/rowW*200 +0.5),1, 200)  
                end  
            end  
        end  
  
    elseif S.activeTab==2 then  
        local fX,fY,rowW=cx+12,cy+12,cw-24  
        local mY=fY+30  
        local btnY=mY+30  
        local sR=btnY+34  
        local sG=sR+30  
        local sB=sG+30  
  
        drawToggle(fX,fY,rowW,"FPS Toggle",S.fpsToggle,      T)  
        drawToggle(fX,mY,rowW,"Watermark", S.watermarkToggle,T)  
  
        drawBox(fX, btnY, rowW, 24, T.PANEL, T.BORDER)  
        drawText(fX+10, btnY+4, T.TEXT, S.rescanning and "Rescan..." or "Rescan")  
  
        drawSlider(fX,sR,rowW,"UI Color (Red)",  S.uiColorR,255,T)  
        drawSlider(fX,sG,rowW,"UI Color (Green)",S.uiColorG,255,T)  
        drawSlider(fX,sB,rowW,"UI Color (Blue)", S.uiColorB,255,T)  
  
        if mousePressed then  
            if     pointInRect(mx,my,fX,fY,rowW,24) then  
                S.fpsToggle=not S.fpsToggle  
                addNotification("FPS Counter "..(S.fpsToggle and "Enabled" or "Disabled"),2)  
            elseif pointInRect(mx,my,fX,mY,rowW,24) then  
                S.watermarkToggle=not S.watermarkToggle  
                addNotification("Watermark "..(S.watermarkToggle and "Enabled" or "Disabled"),2)  
            elseif pointInRect(mx,my,fX,btnY,rowW,24) then  
                runRescan()  
                addNotification(S.pathStatus,2)  
            end  
  
            if pointInRect(mx,my,fX,sR+16,rowW,6) then S.draggingSliderR=true end  
            if pointInRect(mx,my,fX,sG+16,rowW,6) then S.draggingSliderG=true end  
            if pointInRect(mx,my,fX,sB+16,rowW,6) then S.draggingSliderB=true end  
        end  
  
        if mouseDown then  
            local rel=clamp(mx-fX,0,rowW)  
            if     S.draggingSliderR then S.uiColorR=clamp(math.floor(rel/rowW*255+0.5),0,255)  
            elseif S.draggingSliderG then S.uiColorG=clamp(math.floor(rel/rowW*255+0.5),0,255)  
            elseif S.draggingSliderB then S.uiColorB=clamp(math.floor(rel/rowW*255+0.5),0,255)  
            end  
        end  
  
        drawText(fX, cy+ch-14, S.pathColor, S.pathStatus)  
  
    elseif S.activeTab==3 then  
        local w={255,255,255}  
        lbl("Azure Nova "..CFG.VERSION,     w,cx+12,cy+12)  
        lbl("design and tuned with ai help",w,cx+12,cy+32)  
        lbl("UI Built with DXForge",        w,cx+12,cy+52)  
    end  
  
    drawLine(cx+10,cy+ch-20,cx+cw-10,cy+ch-20,T.BORDER)  
end  
  
local function renderNotifications()  
    local now=os.clock()  
    local T=themeFor(S.uiColorR,S.uiColorG,S.uiColorB)  
    local sW=(dx9.size and dx9.size().width)  or 1920  
    local sH=(dx9.size and dx9.size().height) or 1080  
    local yOff=20  
    for i=#S.notifications,1,-1 do  
        local n=S.notifications[i]; local e=now-n.startTime  
        if e>n.duration then table.remove(S.notifications,i)  
        else  
            local bw=string.len(n.text)*7+24; local bh=30  
            local nx=sW-bw-20; local ny=sH-yOff-bh-20  
            drawBox(nx,ny,bw,bh,T.PANEL,T.BORDER)  
            drawText(nx+12,ny+8,T.TEXT,n.text)  
            dx9.DrawFilledBox({nx,ny+bh-2},{nx+bw*(1-e/n.duration),ny+bh},T.ACCENT)  
            yOff=yOff+bh+10  
        end  
    end  
end  
  
-- ============================================================  
-- INPUT  
-- ============================================================  
local key=dx9.GetKey(); local keySig=getKeySig(key)  
local keyPressed  = keySig~="none" and keySig~=S.prevKeySig  
local mouseDown   = dx9.isLeftClickHeld() and true or false  
local mousePressed= mouseDown and not S.prevMouseDown  
local mx,my       = getMouse()  
  
if keyPressed and keyMatches(key,S.menuKey) then  
    S.guiOn=not S.guiOn  
    if S.guiOn then S.activeTab=1 end  
    addNotification("Menu "..(S.guiOn and "Opened" or "Closed"),2)  
end  
if not mouseDown then  
    S.draggingSliderR=false; S.draggingSliderG=false; S.draggingSliderB=false  
    S.draggingSliderSmooth=false; S.draggingSliderDist=false  
    S.draggingSliderSens=false; S.draggingUI=false  
end  
if S.draggingUI then S.menuX=mx-S.dragOffsetX; S.menuY=my-S.dragOffsetY end  
S.prevKeySig=keySig; S.prevMouseDown=mouseDown  
  
-- ============================================================  
-- OVERLAYS  
-- ============================================================  
local T=themeFor(S.uiColorR,S.uiColorG,S.uiColorB)  
local oy=15  
if S.fpsToggle then  
    drawText(15,oy,T.ACCENT,"FPS: "..tostring(_G.AzureNovaFPS.FPS)); oy=oy+18  
end  
if S.watermarkToggle then  
    local txt=CFG.SCRIPT_NAME.." v"..CFG.VERSION  
    local pad=10; local wid=string.len(txt)*7+pad*2  
    local wx=((dx9.size and dx9.size().width) or 1920)/2-wid/2  
    drawBox(wx,15,wid,24,T.PANEL,T.BORDER)  
    drawText(wx+pad,19,T.ACCENT,txt)  
end  
  
drawESP()  
  
-- ============================================================  
-- AIMBOT  
-- ============================================================  
if S.lockOn then  
    local sz=dx9.size(); local sw,sh=sz.width,sz.height  
    local scx=math.floor(sw/2); local scy=math.floor(sh/2)  
  
    dx9.DrawCircle({scx,scy},{255,255,255},math.floor(S.lockOnFov))  
  
    local rHeld=false  
    local ok_rh,rh_val=pcall(dx9.isRightClickHeld)  
    if ok_rh then rHeld=rh_val end  
  
    if rHeld then  
        local ok_dm,game=pcall(dx9.GetDatamodel)  
        if ok_dm and game then  
            local lp=dx9.get_localplayer()  
            if lp then  
                local myChar = getLocalCharacter(lp)  
                local myPos = lp.Position  
  
                if myChar then  
                    local hrp=ffc(myChar,"HumanoidRootPart")  
                    if hrp then  
                        local hp=getPos(hrp)  
                        if hp then myPos=hp end  
                    end  
                end  
  
                if myPos then  
                    local enemyChars = getEnemyCharacters(game, lp)  
  
                    local bestDist = math.huge  
                    local bestPos  = nil  
  
                    for _,char in next,enemyChars do  
                        local torso = ffc(char,"Torso")  
                                   or ffc(char,"UpperTorso")  
                                   or ffc(char,"HumanoidRootPart")  
                        if torso then  
                            local pos=getPos(torso)  
                            if pos then  
                                local wd=dist3(pos,myPos)  
                                if wd>0 and wd<=S.lockOnDist and wd<bestDist then  
                                    bestDist=wd  
                                    bestPos=pos  
                                end  
                            end  
                        end  
                    end  
  
                    if bestPos then  
                        local sp=w2s(bestPos)  
                        if sp then  
                            local deltaX=sp.x-scx  
                            local deltaY=sp.y-scy  
                            local sd=math.sqrt(deltaX*deltaX+deltaY*deltaY)  
  
                            if sd>2 then  
                                local sensDiv    = clamp(S.lockOnSens,1,200)/100  
                                local smoothFrac = 1/S.lockOnSmooth  
  
                                local moveX=(-deltaX/sensDiv)*smoothFrac  
                                local moveY=(-deltaY/sensDiv)*smoothFrac  
  
                                local mag=math.sqrt(moveX*moveX+moveY*moveY)  
                                if mag>15 then  
                                    moveX=moveX*(15/mag)  
                                    moveY=moveY*(15/mag)  
                                end  
  
                                dx9.MouseMove({moveX,moveY})  
                            end  
                        end  
                    end  
                end  
            end  
        end  
    end  
end  
  
-- ============================================================  
-- RENDER  
-- ============================================================  
renderNotifications()  
if S.guiOn then renderMenu(mx,my,mouseDown,mousePressed) end  
  
if not S.initialized then  
    S.initialized=true  
    print("[Azure Nova UI] Loaded. Press "..keyToText(S.menuKey).." to toggle menu.")  
    addNotification("Azure Nova UI Injected Successfully!",4)  
end
