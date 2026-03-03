-- Black Market OS v2.0 - Windows XP Style
-- Système d'exploitation avec interface graphique style Windows XP
--
-- COMMENT UTILISER DES IMAGES PERSONNALISÉES:
-- 1. Créez vos icônes avec le programme 'paint' dans ComputerCraft
-- 2. Sauvegardez-les en .nfp (ex: "market.nfp")
-- 3. Dans desktopIcons, remplacez imagePath = nil par imagePath = "market.nfp"
-- 4. L'image doit faire environ 8x5 pixels pour bien s'afficher
--
-- Exemple: imagePath = "icons/market.nfp"

local w, h = term.getSize()
local currentScreen = "boot"
local username = ""
local password = ""
local loggedIn = false
local marketItems = {}
local selectedItem = 1
local mouseX, mouseY = 0, 0
local startMenuOpen = false

-- Couleurs Windows XP
local xpBlue = colors.blue
local xpLightBlue = colors.lightBlue
local xpGreen = colors.lime
local xpGray = colors.lightGray
local xpDarkGray = colors.gray

-- Configuration des fonds d'écran
local wallpapers = {
    {name = "Bliss (XP)", colors = {xpLightBlue, xpBlue, xpBlue}},
    {name = "Vert Prairie", colors = {colors.lime, colors.green, colors.green}},
    {name = "Coucher Soleil", colors = {colors.orange, colors.red, colors.red}},
    {name = "Nuit Etoilee", colors = {colors.blue, colors.black, colors.black}},
    {name = "Rose Bonbon", colors = {colors.pink, colors.magenta, colors.purple}},
    {name = "Ocean", colors = {colors.cyan, colors.lightBlue, colors.blue}},
    {name = "Black Market", colors = {colors.black, colors.black, colors.black}, hasEye = true},
}
local currentWallpaper = 7

-- Icônes du bureau style XP (cubes 3D simples)
local desktopIcons = {
    {name = "Marche Noir", x = 2, y = 2, width = 10, height = 6, color = colors.red, action = "market", 
     icon = {" ______  ", "/     /| ", "| MN  |/ ", "|_____|  "}},
    {name = "Gestion", x = 2, y = 9, width = 10, height = 6, color = colors.blue, action = "manage", 
     icon = {" ______  ", "/     /| ", "| G   |/ ", "|_____|  "}},
    {name = "Comptes", x = 14, y = 2, width = 10, height = 6, color = colors.green, action = "accounts", 
     icon = {" ______  ", "/     /| ", "| F   |/ ", "|_____|  "}},
    {name = "Parametres", x = 26, y = 2, width = 10, height = 6, color = colors.purple, action = "settings", 
     icon = {" ______  ", "/     /| ", "| X   |/ ", "|_____|  "}},
}

-- Bouton Démarrer et barre des tâches
local startButton = {x = 1, y = h, width = 12, height = 1, text = " Demarrer"}
local taskbarHeight = 1

-- Fonctions utilitaires
local function clearScreen()
    term.setBackgroundColor(colors.black)
    term.clear()
    term.setCursorPos(1, 1)
end

local function drawBox(x, y, width, height, color)
    term.setBackgroundColor(color)
    for i = 0, height - 1 do
        term.setCursorPos(x, y + i)
        term.write(string.rep(" ", width))
    end
end

local function drawButton(x, y, width, height, text, bgColor, textColor)
    drawBox(x, y, width, height, bgColor)
    term.setTextColor(textColor)
    local textX = x + math.floor((width - #text) / 2)
    local textY = y + math.floor(height / 2)
    term.setCursorPos(textX, textY)
    term.write(text)
end

local function drawBorder(x, y, width, height, color)
    term.setBackgroundColor(color)
    term.setTextColor(color)
    -- Haut
    term.setCursorPos(x, y)
    term.write(string.rep("-", width))
    -- Bas
    term.setCursorPos(x, y + height - 1)
    term.write(string.rep("-", width))
    -- Côtés
    for i = 1, height - 2 do
        term.setCursorPos(x, y + i)
        term.write("|")
        term.setCursorPos(x + width - 1, y + i)
        term.write("|")
    end
end

-- Dessiner une fenêtre style Windows XP
local function drawXPWindow(x, y, width, height, title, active)
    -- Barre de titre avec dégradé bleu XP
    local titleColor = active and xpBlue or xpDarkGray
    drawBox(x, y, width, 2, titleColor)
    
    -- Titre
    term.setBackgroundColor(titleColor)
    term.setTextColor(colors.white)
    term.setCursorPos(x + 2, y)
    term.write(title)
    
    -- Boutons de fenêtre (X)
    term.setCursorPos(x + width - 3, y)
    term.setBackgroundColor(colors.red)
    term.write(" X ")
    
    -- Bordure claire (effet 3D haut et gauche)
    term.setBackgroundColor(colors.white)
    for i = 0, height - 1 do
        term.setCursorPos(x, y + i)
        term.write(" ")
    end
    term.setCursorPos(x, y)
    term.write(string.rep(" ", width))
    
    -- Bordure sombre (effet 3D bas et droite)
    term.setBackgroundColor(xpDarkGray)
    for i = 0, height - 1 do
        term.setCursorPos(x + width - 1, y + i)
        term.write(" ")
    end
    term.setCursorPos(x, y + height - 1)
    term.write(string.rep(" ", width))
    
    -- Fond de la fenêtre
    drawBox(x + 1, y + 2, width - 2, height - 3, xpGray)
end

-- Dessiner le fond d'écran
local function drawWallpaper()
    local wp = wallpapers[currentWallpaper]
    for i = 1, h - 1 do
        local color = wp.colors[1]
        if i < h / 3 then
            color = wp.colors[1]
        elseif i < 2 * h / 3 then
            color = wp.colors[2]
        else
            color = wp.colors[3]
        end
        term.setBackgroundColor(color)
        term.setCursorPos(1, i)
        term.write(string.rep(" ", w))
    end
end

-- Dessiner une icône style XP (avec support images .nfp)
local function drawIconBox(icon)
    -- Récupérer la couleur du fond d'écran à cette position
    local wp = wallpapers[currentWallpaper]
    local bgColor = wp.colors[2]
    if icon.y < h / 3 then
        bgColor = wp.colors[1]
    elseif icon.y < 2 * h / 3 then
        bgColor = wp.colors[2]
    else
        bgColor = wp.colors[3]
    end
    
    -- Si une image .nfp est spécifiée, essayer de la charger
    if icon.imagePath and fs.exists(icon.imagePath) then
        local image = paintutils.loadImage(icon.imagePath)
        if image then
            paintutils.drawImage(image, icon.x + 1, icon.y + 1)
        end
    -- Sinon, utiliser l'icône ASCII art
    elseif icon.icon and type(icon.icon) == "table" then
        for i, line in ipairs(icon.icon) do
            local drawY = icon.y + i - 1
            if drawY <= h - 1 and drawY >= 1 then
                term.setCursorPos(icon.x, drawY)
                term.setBackgroundColor(bgColor)
                term.setTextColor(icon.color)
                term.write(line)
            end
        end
    end
    
    -- Nom de l'icône avec fond du wallpaper
    local nameY = icon.y + icon.height - 1
    if nameY <= h - 1 then
        local textX = icon.x + math.floor((icon.width - #icon.name) / 2)
        
        -- Effacer la ligne avec la couleur du fond
        term.setBackgroundColor(bgColor)
        term.setCursorPos(icon.x, nameY)
        term.write(string.rep(" ", icon.width))
        
        -- Texte
        term.setTextColor(colors.white)
        term.setCursorPos(textX, nameY)
        term.write(icon.name)
    end
end

-- Dessiner un bouton style XP avec effet 3D
local function drawXPButton(x, y, width, height, text, pressed)
    local bgColor = pressed and xpDarkGray or xpGray
    drawBox(x, y, width, height, bgColor)
    
    -- Bordures 3D
    if not pressed then
        -- Bordure claire en haut et à gauche
        term.setBackgroundColor(colors.white)
        term.setCursorPos(x, y)
        term.write(string.rep(" ", width))
        for i = 0, height - 1 do
            term.setCursorPos(x, y + i)
            term.write(" ")
        end
        
        -- Bordure sombre en bas et à droite
        term.setBackgroundColor(colors.gray)
        term.setCursorPos(x + 1, y + height - 1)
        term.write(string.rep(" ", width - 1))
        for i = 1, height - 1 do
            term.setCursorPos(x + width - 1, y + i)
            term.write(" ")
        end
    end
    
    -- Texte du bouton
    term.setBackgroundColor(bgColor)
    term.setTextColor(colors.black)
    local textX = x + math.floor((width - #text) / 2)
    local textY = y + math.floor(height / 2)
    term.setCursorPos(textX, textY)
    term.write(text)
end

-- Obtenir l'heure réelle du PC (avec correction fuseau horaire)
local function getRealTime()
    local epoch = os.epoch("local")
    local seconds = math.floor(epoch / 1000)
    -- Ajouter 1 heure (3600 secondes) pour corriger le fuseau horaire
    seconds = seconds + 3600
    local hours = math.floor((seconds / 3600) % 24)
    local minutes = math.floor((seconds / 60) % 60)
    return string.format("%02d:%02d", hours, minutes)
end

-- Dessiner le fond d'écran
local function drawWallpaper()
    local wp = wallpapers[currentWallpaper]
    for i = 1, h - 1 do
        local color = wp.colors[1]
        if i < h / 3 then
            color = wp.colors[1]
        elseif i < 2 * h / 3 then
            color = wp.colors[2]
        else
            color = wp.colors[3]
        end
        term.setBackgroundColor(color)
        term.setCursorPos(1, i)
        term.write(string.rep(" ", w))
    end
    
    -- Dessiner le logo Samurai (Cyberpunk) en bas à droite si c'est le fond Black Market
    if wp.hasEye then
        local logoX = w - 12
        local logoY = h - 8
        
        -- Logo Samurai blanc et rouge
        local samurai = {
            "⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⡀⠀⠀⠀⠀⠀⢀⠀⠀⠀",
            "⠀⠀⠀⢂⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢠⣶⠛⠁⠀⠀⡆⠀⢀⣼⠀⠀⠀",
            "⠀⠀⠀⢾⠀⠀⠀⠀⠀⢀⢠⠂⠀⠀⠀⠀⣿⣇⡄⠀⠀⣰⠇⠀⠘⠀⠀⠀⠀",
            "⠀⠀⠀⢠⠀⠀⠀⠀⠀⢸⣄⠀⠀⢘⡄⢰⣿⣿⣣⣴⠁⢁⢀⢀⠀⠀⠀⢀⠆",
            "⢠⠀⠀⠘⠧⡀⠀⠰⡄⠈⢻⡄⢸⣿⣿⣿⣿⣿⣳⠋⣠⣷⠘⣸⠀⢠⡇⠈⠀",
            "⠰⣇⠀⢧⢠⠘⣦⠀⠀⡘⣾⣞⣿⣿⣿⣿⡿⢟⣿⣿⢿⣟⢀⠟⡄⠈⡇⠀⠀",
            "⠀⠨⡆⠀⠗⠛⠸⡇⢳⣷⣻⡟⠟⡿⣵⣯⣾⢿⣿⣧⣼⣿⢺⢠⣷⠀⠁⠀⠀",
            "⠰⢸⡇⢀⠈⠳⠶⠽⣆⣻⡟⣿⣇⣿⣿⣿⡏⣵⣿⡿⠛⠛⣾⢸⣣⠀⠀⠀⠀",
            "⠀⠀⢉⣾⡄⡁⣲⠫⢡⠘⣷⣿⡿⣧⢇⢿⣷⣿⠟⣡⠀⠀⣷⣟⡇⠀⠀⠀⠀",
            "⠀⠀⢰⣟⠛⠲⢤⣤⡤⠂⢄⢎⣿⣿⣶⣍⢍⡀⠀⠀⣀⣼⣿⡿⠇⠀⠀⠀⠀",
            "⠀⠀⢦⠙⢌⢭⡷⠾⠶⠶⢂⡲⢨⣿⣷⣝⠾⣝⡲⠶⢚⣫⣿⡃⢮⢢⠀⠀⠀",
            "⠀⠀⠈⠳⠀⠀⠀⠀⣤⣼⠷⣗⠘⣿⣿⠛⡳⠉⠻⢿⣿⢵⣮⡛⢤⣃⠇⠀⠀",
            "⠀⠀⠀⠀⠈⠀⠀⢈⣬⡝⣻⣛⠄⢛⣃⣐⡴⣭⣍⠳⡅⠀⠉⠲⠶⠟⠀⠀⠀",
            "⠀⠀⠀⠀⣀⣤⣶⡿⠛⢁⠀⠀⠀⠀⠀⠀⠀⡌⠛⠿⣿⣦⣄⡀⠀⠀⠀⠀⠀",
            "⠀⠀⠀⠀⠀⠘⢿⣤⣘⣷⠶⠚⠛⠻⢶⣭⣁⣴⠖⠀⠀⠀⠀⠀⠀⠀⠀⠀",
            "⠀⠀⠀⠀⠀⠀⠀⠀⠈⠙⠞⣼⣿⣾⣿⣞⠖⠉⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀",
            "⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠈⠉⠈⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀"
        }
        
        for i, line in ipairs(samurai) do
            term.setCursorPos(logoX, logoY + i - 1)
            term.setBackgroundColor(colors.black)
            if i <= 4 then
                term.setTextColor(colors.white)
            else
                term.setTextColor(colors.red)
            end
            term.write(line)
        end
    end
end

-- Dessiner la barre des tâches Windows XP
local function drawTaskbar()
    -- Fond de la barre des tâches (gris clair style XP)
    drawBox(1, h, w, 1, xpGray)
    
    -- Bouton Démarrer avec logo Windows
    term.setBackgroundColor(xpGreen)
    term.setTextColor(colors.white)
    term.setCursorPos(1, h)
    term.write(" ")
    term.setBackgroundColor(xpGray)
    term.setTextColor(colors.black)
    term.write(" Demarrer ")
    
    -- Séparateur
    term.setBackgroundColor(xpDarkGray)
    term.setCursorPos(12, h)
    term.write(" ")
    
    -- Zone des applications
    term.setBackgroundColor(xpGray)
    term.setTextColor(colors.black)
    if loggedIn then
        term.setCursorPos(14, h)
        term.write(" Black Market OS ")
    end
    
    -- Horloge système réelle (à droite)
    local time = getRealTime()
    term.setCursorPos(w - #time - 2, h)
    term.setBackgroundColor(xpLightBlue)
    term.setTextColor(colors.black)
    term.write(" " .. time .. " ")
end

-- Dessiner le menu Démarrer
local function drawStartMenu()
    local menuWidth = 20
    local menuHeight = 10
    local menuX = 1
    local menuY = h - menuHeight - 1
    
    -- Fond du menu avec bordure
    drawBox(menuX, menuY, menuWidth, menuHeight, xpGray)
    
    -- Barre latérale verte Windows XP
    drawBox(menuX, menuY, 3, menuHeight, xpGreen)
    term.setBackgroundColor(xpGreen)
    term.setTextColor(colors.white)
    term.setCursorPos(menuX, menuY + 1)
    term.write(" W ")
    term.setCursorPos(menuX, menuY + 2)
    term.write(" i ")
    term.setCursorPos(menuX, menuY + 3)
    term.write(" n ")
    term.setCursorPos(menuX, menuY + 4)
    term.write(" X ")
    term.setCursorPos(menuX, menuY + 5)
    term.write(" P ")
    
    -- Options du menu
    local menuItems = {
        {text = "Marche Noir", action = "market"},
        {text = "Gestion", action = "manage"},
        {text = "Comptes", action = "accounts"},
        {text = "Parametres", action = "settings"},
        {text = "-------------", action = "none"},
        {text = "Deconnexion", action = "logout"}
    }
    
    for i, item in ipairs(menuItems) do
        term.setBackgroundColor(xpGray)
        term.setTextColor(colors.black)
        term.setCursorPos(menuX + 4, menuY + i - 1)
        if item.text == "-------------" then
            term.write("------------")
        else
            term.write(" " .. item.text)
        end
    end
    
    return menuItems, menuX + 4, menuY
end

local function centerText(y, text, fg, bg)
    term.setTextColor(fg or colors.white)
    term.setBackgroundColor(bg or colors.black)
    local x = math.floor((w - #text) / 2)
    term.setCursorPos(x, y)
    term.write(text)
end

local function isInside(x, y, box)
    return x >= box.x and x < box.x + box.width and y >= box.y and y < box.y + box.height
end

-- Logo ASCII
local function drawLogo()
    local logo = {
        " ____    _    ____  _  __",
        "|  _ \\  / \\  |  _ \\| |/ /",
        "| | | |/ _ \\ | |_) | ' / ",
        "| |_| / ___ \\|  _ <| . \\ ",
        "|____/_/   \\_\\_| \\_\\_|\\_\\",
        "",
        "     ___  ____  ",
        "    / _ \\/ ___| ",
        "   | | | \\___ \\ ",
        "   | |_| |___) |",
        "    \\___/|____/ ",
        "",
        "  -= Marche Noir =-"
    }
    
    for i, line in ipairs(logo) do
        if i <= 5 then
            centerText(3 + i, line, colors.red)
        elseif i <= 11 then
            centerText(3 + i, line, colors.red)
        else
            centerText(3 + i, line, colors.gray)
        end
    end
end

-- Écran de démarrage style Windows XP
local function bootScreen()
    clearScreen()
    
    -- Fond noir avec logo Windows XP style
    term.setBackgroundColor(colors.black)
    term.clear()
    
    -- Logo Windows XP
    local logo = {
        "  _    _ _           _                   ",
        " | |  | (_)         | |                  ",
        " | |  | |_ _ __   __| | _____      _____ ",
        " | |/\\| | | '_ \\ / _` |/ _ \\ \\ /\\ / / __|",
        " \\  /\\  / | | | | (_| | (_) \\ V  V /\\__ \\",
        "  \\/  \\/|_|_| |_|\\__,_|\\___/ \\_/\\_/ |___/",
        "",
        "              X P   S t y l e             "
    }
    
    for i, line in ipairs(logo) do
        term.setTextColor(colors.lightBlue)
        if i == #logo then
            term.setTextColor(colors.lime)
        end
        centerText(4 + i, line)
    end
    
    -- Barre de chargement Windows XP
    centerText(15, "Demarrage de Black Market OS...", colors.white)
    
    local barWidth = 30
    local barX = math.floor((w - barWidth) / 2)
    
    -- Cadre de la barre
    term.setBackgroundColor(colors.gray)
    term.setCursorPos(barX, 17)
    term.write(string.rep(" ", barWidth))
    
    -- Animation de chargement
    for i = 1, barWidth do
        term.setBackgroundColor(colors.lime)
        term.setCursorPos(barX + i - 1, 17)
        term.write(" ")
        sleep(0.02)
    end
    
    sleep(0.3)
    centerText(19, "Appuyez sur une touche...", colors.lightGray)
    os.pullEvent("key")
    currentScreen = "login"
end

-- Écran de connexion style Windows XP
local function loginScreen()
    clearScreen()
    
    -- Fond bleu Windows XP
    for i = 1, h do
        term.setBackgroundColor(xpBlue)
        term.setCursorPos(1, i)
        term.write(string.rep(" ", w))
    end
    
    -- Logo Windows au centre
    centerText(5, "Windows XP", colors.white, xpBlue)
    centerText(6, "Black Market Edition", colors.lightGray, xpBlue)
    
    -- Fenêtre de connexion style XP
    local winX = math.floor(w/2) - 18
    local winY = 9
    drawXPWindow(winX, winY, 36, 12, "Bienvenue", true)
    
    term.setBackgroundColor(xpGray)
    term.setTextColor(colors.black)
    
    -- Message de bienvenue
    term.setCursorPos(winX + 8, winY + 3)
    term.write("Connexion au systeme")
    
    -- Champ utilisateur
    term.setCursorPos(winX + 3, winY + 5)
    term.write("Utilisateur:")
    drawBox(winX + 3, winY + 6, 30, 1, colors.white)
    term.setBackgroundColor(colors.white)
    term.setTextColor(colors.black)
    term.setCursorPos(winX + 4, winY + 6)
    username = read()
    
    -- Champ mot de passe
    term.setBackgroundColor(xpGray)
    term.setCursorPos(winX + 3, winY + 8)
    term.write("Mot de passe:")
    drawBox(winX + 3, winY + 9, 30, 1, colors.white)
    term.setBackgroundColor(colors.white)
    term.setTextColor(colors.black)
    term.setCursorPos(winX + 4, winY + 9)
    password = read("*")
    
    -- Vérification
    if username ~= "" and password ~= "" then
        term.setBackgroundColor(xpGray)
        term.setTextColor(xpGreen)
        term.setCursorPos(winX + 10, winY + 11)
        term.write("Connexion reussie!")
        sleep(1)
        loggedIn = true
        currentScreen = "desktop"
    else
        term.setBackgroundColor(xpGray)
        term.setTextColor(colors.red)
        term.setCursorPos(winX + 8, winY + 11)
        term.write("Identifiants invalides")
        sleep(2)
    end
end

-- Bureau style Windows XP
local function desktopScreen()
    clearScreen()
    
    -- Fond d'écran personnalisable
    drawWallpaper()
    
    -- Dessiner les icônes du bureau
    for _, icon in ipairs(desktopIcons) do
        drawIconBox(icon)
    end
    
    -- Barre des tâches
    drawTaskbar()
    
    -- Attendre un clic
    while true do
        local event, button, x, y = os.pullEvent("mouse_click")
        
        -- Clic sur le bouton Démarrer
        if x >= 1 and x <= 12 and y == h then
            startMenuOpen = not startMenuOpen
            
            if startMenuOpen then
                local menuItems, menuX, menuY = drawStartMenu()
                
                -- Attendre un clic dans le menu
                while startMenuOpen do
                    local event2, button2, x2, y2 = os.pullEvent("mouse_click")
                    
                    -- Vérifier clic sur une option du menu
                    for i, item in ipairs(menuItems) do
                        if x2 >= menuX and x2 < menuX + 16 and y2 == menuY + i - 1 then
                            if item.action == "logout" then
                                loggedIn = false
                                currentScreen = "login"
                                return
                            elseif item.action ~= "none" then
                                currentScreen = item.action
                                return
                            end
                        end
                    end
                    
                    -- Clic en dehors du menu = fermer
                    if x2 < menuX or x2 >= menuX + 20 or y2 < menuY or y2 >= menuY + 10 then
                        startMenuOpen = false
                        desktopScreen()
                        return
                    end
                end
            end
        end
        
        -- Vérifier clic sur icônes
        for _, icon in ipairs(desktopIcons) do
            if isInside(x, y, icon) then
                currentScreen = icon.action
                return
            end
        end
    end
end

-- Marché noir
local function marketScreen()
    clearScreen()
    drawBox(1, 1, w, 2, colors.gray)
    term.setBackgroundColor(colors.gray)
    term.setTextColor(colors.red)
    term.setCursorPos(2, 1)
    term.write("$ MARCHE NOIR $")
    term.setTextColor(colors.white)
    term.setCursorPos(2, 2)
    term.write("Catalogue des articles")
    
    if #marketItems == 0 then
        -- Cadre vide stylisé
        drawBox(5, 6, w - 10, 6, colors.gray)
        term.setBackgroundColor(colors.gray)
        centerText(8, "Aucun article disponible", colors.yellow)
        centerText(10, "Utilisez Gestion pour ajouter", colors.white)
    else
        local startY = 4
        for i, item in ipairs(marketItems) do
            local y = startY + (i - 1) * 3
            if y < h - 3 then
                -- Cadre pour chaque article
                drawBox(2, y, w - 4, 2, colors.gray)
                
                term.setBackgroundColor(colors.gray)
                term.setTextColor(colors.white)
                term.setCursorPos(4, y)
                term.write(string.format("[%d] %s", i, item.name))
                
                term.setCursorPos(w - 30, y)
                term.setTextColor(colors.yellow)
                term.write("$ " .. item.price)
                
                term.setCursorPos(w - 18, y)
                term.setTextColor(colors.lime)
                term.write("Stock: " .. item.stock)
                
                term.setCursorPos(4, y + 1)
                term.setTextColor(colors.lightGray)
                term.write("Categorie: " .. (item.category or "N/A"))
            end
        end
    end
    
    -- Boutons
    drawButton(2, h - 1, 10, 1, "< Retour", colors.red, colors.white)
    drawButton(w - 15, h - 1, 14, 1, "Actualiser", colors.blue, colors.white)
    
    while true do
        local event, button, x, y = os.pullEvent("mouse_click")
        if x >= 2 and x < 12 and y == h - 1 then
            currentScreen = "desktop"
            return
        end
    end
end

-- Marché noir dans une fenêtre XP
local function marketScreen()
    clearScreen()
    
    -- Fond d'écran
    drawWallpaper()
    
    -- Fenêtre du marché
    local winX = 3
    local winY = 2
    local winW = w - 6
    local winH = h - 4
    
    drawXPWindow(winX, winY, winW, winH, "Marche Noir - Catalogue", true)
    
    term.setBackgroundColor(xpGray)
    
    if #marketItems == 0 then
        term.setTextColor(colors.black)
        centerText(math.floor(h/2), "Aucun article disponible", colors.black, xpGray)
        centerText(math.floor(h/2) + 2, "Utilisez Gestion pour ajouter des articles", colors.gray, xpGray)
    else
        -- En-têtes de colonnes
        term.setBackgroundColor(xpDarkGray)
        term.setTextColor(colors.white)
        term.setCursorPos(winX + 2, winY + 3)
        term.write(" Article")
        term.setCursorPos(winX + 25, winY + 3)
        term.write("Prix")
        term.setCursorPos(winX + 35, winY + 3)
        term.write("Stock")
        term.setCursorPos(winX + 45, winY + 3)
        term.write("Categorie")
        
        -- Liste des articles
        local startY = winY + 4
        for i, item in ipairs(marketItems) do
            local y = startY + (i - 1) * 2
            if y < winY + winH - 4 then
                -- Ligne alternée
                local bgColor = (i % 2 == 0) and colors.white or xpGray
                drawBox(winX + 2, y, winW - 4, 1, bgColor)
                
                term.setBackgroundColor(bgColor)
                term.setTextColor(colors.black)
                term.setCursorPos(winX + 2, y)
                term.write(string.format("%d. %s", i, item.name))
                
                term.setCursorPos(winX + 25, y)
                term.setTextColor(colors.orange)
                term.write("$" .. item.price)
                
                term.setCursorPos(winX + 35, y)
                term.setTextColor(xpGreen)
                term.write(item.stock)
                
                term.setCursorPos(winX + 45, y)
                term.setTextColor(colors.blue)
                term.write(item.category or "N/A")
            end
        end
    end
    
    -- Boutons en bas de la fenêtre
    drawXPButton(winX + 2, winY + winH - 3, 12, 2, "Fermer", false)
    drawXPButton(winX + winW - 16, winY + winH - 3, 14, 2, "Actualiser", false)
    
    -- Barre des tâches
    drawTaskbar()
    
    while true do
        local event, button, x, y = os.pullEvent("mouse_click")
        
        -- Bouton Fermer ou X
        if (x >= winX + 2 and x < winX + 14 and y >= winY + winH - 3 and y < winY + winH - 1) or
           (x >= winX + winW - 3 and x <= winX + winW - 1 and y == winY) then
            currentScreen = "desktop"
            return
        end
        
        -- Clic sur Démarrer
        if x >= 1 and x <= 12 and y == h then
            currentScreen = "desktop"
            return
        end
    end
end
-- Gestion des articles dans une fenêtre XP
local function manageScreen()
    clearScreen()
    
    -- Fond d'écran
    drawWallpaper()
    
    -- Fenêtre de gestion
    local winX = 3
    local winY = 2
    local winW = w - 6
    local winH = h - 4
    
    drawXPWindow(winX, winY, winW, winH, "Gestion - Articles", true)
    
    term.setBackgroundColor(xpGray)
    
    -- Boutons d'action
    local addBtn = {x = winX + 5, y = winY + 4, width = 16, height = 3}
    local delBtn = {x = winX + 25, y = winY + 4, width = 16, height = 3}
    
    drawXPButton(addBtn.x, addBtn.y, addBtn.width, addBtn.height, "Ajouter", false)
    drawXPButton(delBtn.x, delBtn.y, delBtn.width, delBtn.height, "Supprimer", false)
    
    -- Liste des articles
    if #marketItems > 0 then
        term.setBackgroundColor(xpGray)
        term.setTextColor(colors.black)
        term.setCursorPos(winX + 5, winY + 9)
        term.write("Articles actuels (" .. #marketItems .. "):")
        
        -- Cadre de liste
        drawBox(winX + 5, winY + 10, winW - 10, winH - 14, colors.white)
        
        for i, item in ipairs(marketItems) do
            if winY + 10 + i < winY + winH - 5 then
                term.setBackgroundColor(colors.white)
                term.setTextColor(colors.black)
                term.setCursorPos(winX + 6, winY + 10 + i)
                term.write(string.format("%d. %s", i, item.name))
                term.setTextColor(colors.orange)
                term.setCursorPos(winX + 30, winY + 10 + i)
                term.write("$" .. item.price)
                term.setTextColor(xpGreen)
                term.setCursorPos(winX + 40, winY + 10 + i)
                term.write("x" .. item.stock)
            end
        end
    else
        term.setBackgroundColor(xpGray)
        term.setTextColor(colors.gray)
        centerText(winY + 12, "Aucun article", colors.gray, xpGray)
    end
    
    -- Bouton Fermer
    drawXPButton(winX + 2, winY + winH - 3, 12, 2, "Fermer", false)
    
    -- Barre des tâches
    drawTaskbar()
    
    while true do
        local event, button, x, y = os.pullEvent("mouse_click")
        
        if isInside(x, y, addBtn) then
            currentScreen = "addItem"
            return
        elseif isInside(x, y, delBtn) then
            currentScreen = "deleteItem"
            return
        elseif (x >= winX + 2 and x < winX + 14 and y >= winY + winH - 3) or
               (x >= winX + winW - 3 and x <= winX + winW - 1 and y == winY) then
            currentScreen = "desktop"
            return
        end
        
        if x >= 1 and x <= 12 and y == h then
            currentScreen = "desktop"
            return
        end
    end
end

-- Ajouter un article dans une fenêtre XP
local function addItemScreen()
    clearScreen()
    
    -- Fond d'écran
    drawWallpaper()
    
    -- Fenêtre
    local winX = 8
    local winY = 4
    local winW = w - 16
    local winH = h - 8
    
    drawXPWindow(winX, winY, winW, winH, "Ajouter un article", true)
    
    term.setBackgroundColor(xpGray)
    term.setTextColor(colors.black)
    
    -- Formulaire
    term.setCursorPos(winX + 3, winY + 4)
    term.write("Nom de l'article:")
    drawBox(winX + 3, winY + 5, winW - 6, 1, colors.white)
    term.setBackgroundColor(colors.white)
    term.setCursorPos(winX + 4, winY + 5)
    local itemName = read()
    
    term.setBackgroundColor(xpGray)
    term.setCursorPos(winX + 3, winY + 7)
    term.write("Prix:")
    drawBox(winX + 3, winY + 8, 15, 1, colors.white)
    term.setBackgroundColor(colors.white)
    term.setCursorPos(winX + 4, winY + 8)
    local itemPrice = tonumber(read()) or 0
    
    term.setBackgroundColor(xpGray)
    term.setCursorPos(winX + 3, winY + 10)
    term.write("Stock:")
    drawBox(winX + 3, winY + 11, 15, 1, colors.white)
    term.setBackgroundColor(colors.white)
    term.setCursorPos(winX + 4, winY + 11)
    local itemStock = tonumber(read()) or 0
    
    term.setBackgroundColor(xpGray)
    term.setCursorPos(winX + 3, winY + 13)
    term.write("Categorie:")
    drawBox(winX + 3, winY + 14, winW - 6, 1, colors.white)
    term.setBackgroundColor(colors.white)
    term.setCursorPos(winX + 4, winY + 14)
    local itemCategory = read()
    
    if itemName ~= "" then
        table.insert(marketItems, {
            name = itemName,
            price = itemPrice,
            stock = itemStock,
            category = itemCategory
        })
        
        term.setBackgroundColor(xpGray)
        term.setTextColor(xpGreen)
        term.setCursorPos(winX + 3, winY + 16)
        term.write("Article ajoute avec succes!")
        sleep(1.5)
    else
        term.setBackgroundColor(xpGray)
        term.setTextColor(colors.red)
        term.setCursorPos(winX + 3, winY + 16)
        term.write("Nom invalide!")
        sleep(1.5)
    end
    
    currentScreen = "manage"
end

-- Supprimer un article dans une fenêtre XP
local function deleteItemScreen()
    clearScreen()
    
    -- Fond d'écran
    drawWallpaper()
    
    if #marketItems == 0 then
        -- Message d'erreur dans une fenêtre
        local winX = 15
        local winY = 8
        drawXPWindow(winX, winY, 30, 8, "Erreur", true)
        term.setBackgroundColor(xpGray)
        term.setTextColor(colors.red)
        centerText(winY + 4, "Aucun article", colors.red, xpGray)
        centerText(winY + 5, "a supprimer", colors.red, xpGray)
        sleep(2)
        currentScreen = "manage"
        return
    end
    
    -- Fenêtre
    local winX = 5
    local winY = 3
    local winW = w - 10
    local winH = h - 6
    
    drawXPWindow(winX, winY, winW, winH, "Supprimer un article", true)
    
    term.setBackgroundColor(xpGray)
    term.setTextColor(colors.black)
    term.setCursorPos(winX + 3, winY + 3)
    term.write("Selectionnez l'article a supprimer:")
    
    -- Liste des articles
    drawBox(winX + 3, winY + 5, winW - 6, winH - 10, colors.white)
    
    for i, item in ipairs(marketItems) do
        if winY + 5 + i < winY + winH - 6 then
            term.setBackgroundColor(colors.white)
            term.setTextColor(colors.black)
            term.setCursorPos(winX + 4, winY + 5 + i)
            term.write(string.format("%d. %s (Prix: %d, Stock: %d)", i, item.name, item.price, item.stock))
        end
    end
    
    term.setBackgroundColor(xpGray)
    term.setTextColor(colors.black)
    term.setCursorPos(winX + 3, winY + winH - 5)
    term.write("Numero:")
    drawBox(winX + 12, winY + winH - 5, 10, 1, colors.white)
    term.setBackgroundColor(colors.white)
    term.setCursorPos(winX + 13, winY + winH - 5)
    local choice = tonumber(read())
    
    if choice and choice >= 1 and choice <= #marketItems then
        table.remove(marketItems, choice)
        term.setBackgroundColor(xpGray)
        term.setTextColor(xpGreen)
        term.setCursorPos(winX + 3, winY + winH - 3)
        term.write("Article supprime!")
        sleep(1.5)
    else
        term.setBackgroundColor(xpGray)
        term.setTextColor(colors.red)
        term.setCursorPos(winX + 3, winY + winH - 3)
        term.write("Numero invalide!")
        sleep(1.5)
    end
    
    currentScreen = "manage"
end

-- Comptes et statistiques dans une fenêtre XP
local function accountsScreen()
    clearScreen()
    
    -- Fond d'écran
    drawWallpaper()
    
    -- Fenêtre
    local winX = 3
    local winY = 2
    local winW = w - 6
    local winH = h - 4
    
    drawXPWindow(winX, winY, winW, winH, "Comptes - Statistiques", true)
    
    term.setBackgroundColor(xpGray)
    
    -- Calcul des statistiques
    local totalItems = #marketItems
    local totalValue = 0
    local totalStock = 0
    
    for _, item in ipairs(marketItems) do
        totalValue = totalValue + (item.price * item.stock)
        totalStock = totalStock + item.stock
    end
    
    -- Panneau de statistiques
    drawBox(winX + 5, winY + 4, 20, 6, colors.white)
    term.setBackgroundColor(colors.white)
    term.setTextColor(xpBlue)
    term.setCursorPos(winX + 7, winY + 5)
    term.write("STATISTIQUES")
    
    term.setTextColor(colors.black)
    term.setCursorPos(winX + 7, winY + 7)
    term.write("Articles: " .. totalItems)
    term.setCursorPos(winX + 7, winY + 8)
    term.write("Stock total: " .. totalStock)
    
    drawBox(winX + 28, winY + 4, 22, 6, colors.white)
    term.setBackgroundColor(colors.white)
    term.setTextColor(xpGreen)
    term.setCursorPos(winX + 30, winY + 5)
    term.write("VALEUR TOTALE")
    term.setTextColor(colors.orange)
    term.setCursorPos(winX + 33, winY + 7)
    term.write("$ " .. totalValue)
    term.setTextColor(colors.gray)
    term.setCursorPos(winX + 35, winY + 8)
    term.write("Moet")
    
    -- Top 3 articles
    if #marketItems > 0 then
        term.setBackgroundColor(xpGray)
        term.setTextColor(colors.black)
        term.setCursorPos(winX + 5, winY + 12)
        term.write("TOP 3 ARTICLES LES PLUS CHERS")
        
        drawBox(winX + 5, winY + 13, winW - 10, 8, colors.white)
        
        local sorted = {}
        for _, item in ipairs(marketItems) do
            table.insert(sorted, item)
        end
        table.sort(sorted, function(a, b) return a.price > b.price end)
        
        for i = 1, math.min(3, #sorted) do
            local medals = {"[OR]", "[AG]", "[BR]"}
            term.setBackgroundColor(colors.white)
            term.setTextColor(colors.orange)
            term.setCursorPos(winX + 7, winY + 13 + i)
            term.write(medals[i])
            term.setTextColor(colors.black)
            term.write(" " .. sorted[i].name)
            term.setTextColor(colors.orange)
            term.setCursorPos(winX + winW - 18, winY + 13 + i)
            term.write("$ " .. sorted[i].price)
        end
    end
    
    -- Bouton Fermer
    drawXPButton(winX + 2, winY + winH - 3, 12, 2, "Fermer", false)
    
    -- Barre des tâches
    drawTaskbar()
    
    while true do
        local event, button, x, y = os.pullEvent("mouse_click")
        if (x >= winX + 2 and x < winX + 14 and y >= winY + winH - 3) or
           (x >= winX + winW - 3 and x <= winX + winW - 1 and y == winY) then
            currentScreen = "desktop"
            return
        end
        if x >= 1 and x <= 12 and y == h then
            currentScreen = "desktop"
            return
        end
    end
end



-- Paramètres avec sélection de fond d'écran
local function settingsScreen()
    clearScreen()
    
    -- Fond d'écran
    drawWallpaper()
    
    -- Fenêtre
    local winX = 8
    local winY = 3
    local winW = w - 16
    local winH = h - 6
    
    drawXPWindow(winX, winY, winW, winH, "Parametres systeme", true)
    
    term.setBackgroundColor(xpGray)
    
    -- Icône système
    drawBox(winX + 3, winY + 4, 8, 6, xpBlue)
    term.setBackgroundColor(xpBlue)
    term.setTextColor(colors.white)
    term.setCursorPos(winX + 5, winY + 6)
    term.write("XP")
    
    -- Informations système
    term.setBackgroundColor(xpGray)
    term.setTextColor(colors.black)
    term.setCursorPos(winX + 13, winY + 5)
    term.write("Black Market OS")
    term.setCursorPos(winX + 13, winY + 6)
    term.setTextColor(colors.gray)
    term.write("Version 2.0 XP Edition")
    
    -- Séparateur
    term.setBackgroundColor(xpDarkGray)
    term.setCursorPos(winX + 3, winY + 11)
    term.write(string.rep(" ", winW - 6))
    
    -- Détails
    term.setBackgroundColor(xpGray)
    term.setTextColor(colors.black)
    term.setCursorPos(winX + 3, winY + 13)
    term.write("Utilisateur:")
    term.setTextColor(xpBlue)
    term.setCursorPos(winX + 18, winY + 13)
    term.write(username)
    
    term.setTextColor(colors.black)
    term.setCursorPos(winX + 3, winY + 15)
    term.write("Resolution:")
    term.setTextColor(xpBlue)
    term.setCursorPos(winX + 18, winY + 15)
    term.write(w .. " x " .. h)
    
    term.setTextColor(colors.black)
    term.setCursorPos(winX + 3, winY + 17)
    term.write("Heure:")
    term.setTextColor(xpBlue)
    term.setCursorPos(winX + 18, winY + 17)
    term.write(getRealTime())
    
    -- Section fond d'écran
    term.setBackgroundColor(xpDarkGray)
    term.setCursorPos(winX + 3, winY + 19)
    term.write(string.rep(" ", winW - 6))
    
    term.setBackgroundColor(xpGray)
    term.setTextColor(colors.black)
    term.setCursorPos(winX + 3, winY + 20)
    term.write("Fond d'ecran:")
    term.setTextColor(xpBlue)
    term.setCursorPos(winX + 18, winY + 20)
    term.write(wallpapers[currentWallpaper].name)
    
    -- Boutons
    drawXPButton(winX + 2, winY + winH - 3, 10, 2, "OK", false)
    drawXPButton(winX + 14, winY + winH - 3, 14, 2, "Changer FE", false)
    drawXPButton(winX + 30, winY + winH - 3, 12, 2, "Annuler", false)
    
    -- Barre des tâches
    drawTaskbar()
    
    while true do
        local event, button, x, y = os.pullEvent("mouse_click")
        
        -- Bouton Changer Fond d'Écran
        if x >= winX + 14 and x < winX + 28 and y >= winY + winH - 3 and y < winY + winH - 1 then
            currentWallpaper = currentWallpaper + 1
            if currentWallpaper > #wallpapers then
                currentWallpaper = 1
            end
            settingsScreen()
            return
        end
        
        -- Bouton OK ou Annuler ou X
        if (x >= winX + 2 and x < winX + 12 and y >= winY + winH - 3) or
           (x >= winX + 30 and x < winX + 42 and y >= winY + winH - 3) or
           (x >= winX + winW - 3 and x <= winX + winW - 1 and y == winY) then
            currentScreen = "desktop"
            return
        end
        
        -- Clic sur Démarrer
        if x >= 1 and x <= 12 and y == h then
            currentScreen = "desktop"
            return
        end
    end
end

-- Boucle principale
local function main()
    while true do
        if currentScreen == "boot" then
            bootScreen()
        elseif currentScreen == "login" then
            loginScreen()
        elseif currentScreen == "desktop" and loggedIn then
            desktopScreen()
        elseif currentScreen == "market" then
            marketScreen()
        elseif currentScreen == "manage" then
            manageScreen()
        elseif currentScreen == "addItem" then
            addItemScreen()
        elseif currentScreen == "deleteItem" then
            deleteItemScreen()
        elseif currentScreen == "accounts" then
            accountsScreen()
        elseif currentScreen == "settings" then
            settingsScreen()
        end
    end
end

-- Démarrage
main()
