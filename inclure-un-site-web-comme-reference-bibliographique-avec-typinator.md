# Inclure un site web comme référence bibliographique avec Typinator

Voici le snippet `siteweb`  construit dans Typinator :

`{Scripts/BrowserPageInfo.scpt auto,url}, raccourci : {Scripts/TinyURL.applescript {Scripts/BrowserPageInfo.scpt auto,url}}, "{Scripts/BrowserPageInfo.scpt auto,title}", page web consultée le {D} {NN} {YYYY} à {h024}:{m}.`

Il fait référence au script suivant \(**merci à Johny Thompson pour la base de ce script**\) :

```text
-- "BrowserPageInfo" Library: Returns the URL and/or Title of the currently active tab in your web browser. It supports Safari, Google Chrome, Google Chrome Canary, Chromium and Webkit. --The first parameter is the name of the browser from which to fetch information about the current URL.
--The second parameter can be "url" to get the page URL, "title" for its title, or "both" for a combination using Markdown syntax.
-- Power-user notes: -- * If you set the browser name to "auto", such as "auto,url", it will check if the currently active (topmost) OS X window is a supported browser and use that, or if that fails it will select the first supported browser found in the list of running processes. This mode can be very useful if you use multiple browsers and don't know which one you'll be running at any moment.
-- * There's a hidden 3rd parameter for enabling power-user flags. The 3rd parameter must start with a plus sign followed by individual letters for each flag you want to turn on (such as "+tymc"). Currently the only available flags are "t" and "s", and if you want to enable both of them you would use "Safari,url,+ts" as the script parameter. See the detailed descriptions of the current flags (and any future flags) in the source code below, in the 'Flag "x": ...' comments. 
-- Version 1.1, (C) Johnny Thompson, 2016-05-03
-- Feel free to modify the script for your own use, but leave the copyright notice intact. 
-- If you would like to help us add support for other browsers, we welcome you to submit their necessary AppleScript commands to us (see the "allSupportedBrowsers" array in the code below for the entry format we'll need for your browser). Note that we cannot support Firefox, since they do not support getting the URL or Title via AppleScript.
on splitString(theText, theDelimiter) local ASTID, theText, theDelimiter, lst set ASTID to AppleScript's text item delimiters try considering case set AppleScript's text item delimiters to theDelimiter set lst to every text item of theText end considering set AppleScript's text item delimiters to ASTID return lst on error eMsg number eNum set AppleScript's text item delimiters to ASTID error "Can't splitString: " & eMsg number eNum end try end splitString
on replaceString(theText, oldString, newString) local ASTID, theText, oldString, newString, lst set ASTID to AppleScript's text item delimiters try considering case set AppleScript's text item delimiters to oldString set lst to every text item of theText set AppleScript's text item delimiters to newString set theText to lst as string end considering set AppleScript's text item delimiters to ASTID return theText on error eMsg number eNum set AppleScript's text item delimiters to ASTID error "Can't replaceString: " & eMsg number eNum end try end replaceString
on silenceHelper(silence, txt) -- Makes it easy and clean to support message silencing, by passing "true" as the first parameter. if (silence is true) then return "" else return txt end if end silenceHelper
on expand(rawArgs) -- parameter: Safari,url/title/both -- Determine what browser to check and whether to get the url or title or both. if (class of rawArgs is not string) then return "'You must give this script a single string parameter'" end if set args to my splitString(rawArgs, ",") if ((count of args) < 2) then return "'The script needs at least two comma-separated parameters'" end if set theBrowser to item 1 of args set wanted to item 2 of args

-- Validate the "wanted" parameter.
if (wanted is not "url") and (wanted is not "title") and (wanted is not "both") then
    return "'Invalid second argument, must be one of these: url, title, both.'"
end if

-- Look for power-user flags (via a third hidden argument which must start with a plus sign followed by individual letters for each flag, such as "Safari,url,+t"; if it doesn't start with a plus, it's ignored)
set flags_mustBeTopmost to false
set flags_silenceErrors to false
if ((count of args) ≥ 3) then
    set flagargs to item 3 of args
    if ((offset of "+" in flagargs) is 1) then
        -- It starts with a plus like a valid flag parameter, so look for all supported flags.
        considering case
            -- Flag "t": The desired browser must be running and must be the current topmost/active window, otherwise the script will abort with an error message.
            if ((offset of "t" in flagargs) is not 0) then
                set flags_mustBeTopmost to true
            end if
            -- Flag "s": Silences *all* error messages, so that there's never any output if a browser is missing, not running, not topmost, or if there's an AppleScript error. The only error messages that will get through are when the parameters to the script are invalid, since parameter count and values are checked before flags are processed. This means that if you've written the parameters correctly (which you should only have to do once), then the only script output will be URLs/Titles, and only if all conditions are perfect, otherwise it outputs nothing at all.
            if ((offset of "s" in flagargs) is not 0) then
                set flags_silenceErrors to true
            end if
        end considering
    end if
end if

try
    -- The current list of supported browsers. Easy to extend in the future.
    set allSupportedBrowsers to {¬
        {appName:"Safari", cmds:{cmdURL:"URL of front document", cmdTitle:"name of front document"}}, ¬
        {appName:"Google Chrome", cmds:{cmdURL:"URL of active tab of front window", cmdTitle:"title of active tab of front window"}}, ¬
        {appName:"Google Chrome Canary", cmds:{cmdURL:"URL of active tab of front window", cmdTitle:"title of active tab of front window"}}, ¬
        {appName:"Chromium", cmds:{cmdURL:"URL of active tab of front window", cmdTitle:"title of active tab of front window"}}, ¬
        {appName:"Webkit", cmds:{cmdURL:"URL of front document", cmdTitle:"name of front document"}} ¬
            }

    -- Get the name of the currently active application (such as "Safari"). It's used by several optional features in this script.
    tell application "System Events" to set activeAppName to name of first application process whose frontmost is true

    -- Resolve the "auto" browser type (if given by the user) by replacing it with the name of the best-matching, currently running, supported browser. If no match is found, we'll abort processing.
    if (theBrowser is "auto") then
        set foundAuto to false
        -- First check if the topmost (currently active) application is one of the supported browsers. If so choose that one, so that people with multiple browsers running simultaneously get the most proper results when one of them is active.
        repeat with anItem in allSupportedBrowsers
            if (activeAppName is (appName of anItem)) then
                set foundAuto to true
                set theBrowser to appName of anItem
                exit repeat
            end if
        end repeat
        -- If the topmost app wasn't one of the supported browsers, we'll now scan all running applications to look for a supported browser, in the order that they're listed in the "allSupportedBrowsers" array.
        if (not foundAuto) then
            -- Get an array with the names of all currently running apps, such as "{"loginwindow", "Dock", "Google Chrome", ...}".
            tell application "System Events" to set runningApps to (name of processes)
            -- Now look for each supported browser in the list of running apps and select the first one we find.
            repeat with anItem in allSupportedBrowsers
                if (runningApps contains (appName of anItem)) then
                    set foundAuto to true
                    set theBrowser to appName of anItem
                    exit repeat
                end if
            end repeat
        end if
        -- If we were unable to resolve "auto" to the name of a supported and currently running browser, then we have to abort here.
        if (not foundAuto) then
            return my silenceHelper(flags_silenceErrors, "'Auto-browser detection could not find any running supported web browser'")
        end if
    end if

    -- Ensure that the browser is supported and get the necessary commands for it.
    set browserCmds to missing value
    set isSupportedBrowser to false
    repeat with anItem in allSupportedBrowsers
        if (theBrowser is (appName of anItem)) then
            set isSupportedBrowser to true
            set browserCmds to cmds of anItem
            exit repeat
        end if
    end repeat
    if (not isSupportedBrowser) then
        return my silenceHelper(flags_silenceErrors, "'The \"" & theBrowser & "\" browser is not supported by this script'")
    end if

    -- Topmost flag handling: If the given/detected browser isn't the currently active OS X window, then just return an error message, or an empty string (if the silence flag is also specified).
    if (flags_mustBeTopmost) and (activeAppName is not theBrowser) then
        return my silenceHelper(flags_silenceErrors, "'The \"" & theBrowser & "\" browser is not the topmost window'")
    end if

    -- Get the bundle name (such as "com.apple.Safari"), to allow us to talk to the browser dynamically in a reliable way. If the browser app bundle can't be automatically found by the system, the user will be prompted by AppleScript to navigate to the application.
    set browserAppId to id of application theBrowser

    -- We'll only talk to the browser if it's running, otherwise AppleScript would cause it to be launched if it was closed, which people usually don't want.
    if application id browserAppId is running then
        set currentTabURL to missing value
        set currentTabTitle to missing value

        -- Wrapping the application calls in "run script" makes those lines compile dynamically on demand.
        -- It allows us to dynamically decide how to talk to each specific browser, without any code repetition.
        if (wanted is "url") or (wanted is "both") then
            set currentTabURL to (run script "tell application id \"" & browserAppId & "\" to return " & cmdURL of browserCmds) as string
        end if
        if (wanted is "title") or (wanted is "both") then
            set currentTabTitle to (run script "tell application id \"" & browserAppId & "\" to return " & cmdTitle of browserCmds) as string
        end if

        -- If they wanted both the title and URL, we'll have to format the resulting string accordingly.
        set finalString to ""
        if (wanted is "both") then
            -- Special handling of Markdown output: We will replace any [ or ] brackets in the title of the page with regular parenthesis, since they would otherwise interfere with the markdown. We'll also replace any parenthesis in the URL itself with their URL-encoded equivalents, since some old/basic Markdown parsers can otherwise misunderstand them as the end of the URL.
            set finalString to ("[" & my replaceString(my replaceString(currentTabTitle, "[", "("), "]", ")") & "](" & my replaceString(my replaceString(currentTabURL, "(", "%28"), ")", "%29") & ")") as string
        else if (wanted is "title") then
            set finalString to currentTabTitle as string
        else if (wanted is "url") then
            set finalString to currentTabURL as string
        end if

        -- We're done!
        return finalString
    else
        return my silenceHelper(flags_silenceErrors, "'The \"" & theBrowser & "\" browser is not running'")
    end if

on error eMsg
    -- This even catches errors generated by the dynamic "run script" commands above, such as if the browser doesn't understand the command.
    return my silenceHelper(flags_silenceErrors, "AppleScript Error [" & eMsg & "]")
end try
end expand
```



