---
layout: post
title: Enabling C# Sonar Analyzers in Visual Studio Code
date: 2023-04-01T00:00:00.000+1000
tags: [.NET, Static Analysis]
# images: ["posts/20230401-enabling-c#-sonar-analysers-in-visual-studio-code/opengraph.png"]
---

The C# Sonar analysers provide immense value in identifying bugs, vulnerabilities and code smells within your code. Here's how you can enable them for use in Visual Studio Code.

<!--more-->

## Enable Roslyn Analyzers

If you haven't already, you'll want to go ahead and enable support for Roslyn analyzers in Visual Studio Code's OmniSharp configuration. You can do this either via the Settings UI (go to `File > Preferences > Settings` or press `Ctrl+,`) &mdash; search for "Omnisharp: Enable Roslyn Analyzers" &mdash; or via the `omnisharp.enableRoslynAnalyzers` value in your Settings JSON (open the Command Palette ([?](https://code.visualstudio.com/docs/getstarted/userinterface#_command-palette)) with `F1` or `Ctrl+Shift+P` and use the `Preferences: Open User Settings (JSON)` command); as below:

``` jsonc
// settings.json
{
    // ...
    "omnisharp.enableRoslynAnalyzers": true
}
```

With Roslyn analyzers enabled, you'll immediately 

## Configure the C# Sonar Analyzers

Until some [official support](https://portal.productboard.com/sonarsource/4-sonarlint/c/188-support-c-in-visual-studio-code) is added &mdash; which at this stage doesn't look to be happening until after Microsoft moves the C# extension away from OmniSharp (see [OmniSharp/omnisharp-vscode#5276](https://github.com/OmniSharp/omnisharp-vscode/issues/5276)) &mdash; C# Sonar analyzers need to be manually configured.

Referring to [OmniSharp's configuration options](https://github.com/OmniSharp/omnisharp-roslyn/wiki/Configuration-Options), OmniSharp will read configuration values from `omnisharp.json` files located 

``` jsonc
// ~\.omnisharp\omnisharp.json
{
    "RoslynExtensionsOptions": {
        "enableAnalyzersSupport": true,
        "LocationPaths": [
            "C:\\Users\\Jacob\\.omnisharp\\SonarAnalyzer.CSharp\\analyzers"
        ]
    }
}
```

# References

- [C# support in VSCode &mdash; Comment #29 by mcm_ham | Sonar Community Forums](https://community.sonarsource.com/t/c-support-in-vscode/5888/29)