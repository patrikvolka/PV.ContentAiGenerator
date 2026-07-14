# PV.ContentAiGenerator
Enterprise-grade AI content generation module for Orchard Core 3.0. Features Google Gemini API integration, AI agent orchestration (text &amp; images), and Syncfusion-powered UI for seamless CMS data mapping.

# Code Highlights

This section highlights the architectural decisions, best practices, and advanced Orchard Core 3.0 integrations implemented in this enterprise-grade module. The project adheres to SOLID principles, heavily utilizes native Orchard Core features (Content Parts, Liquid Templates, Resource Manager), and ensures optimal performance and memory safety.

## 1. Secure Localization & Syncfusion UI Integration (Razor / JS)

This snippet demonstrates the best practice for localizing static JavaScript files within Orchard Core. Instead of hardcoding `@T["..."]` directly into JS (which can break syntax due to unescaped quotes), we serialize a C# `Dictionary` to JSON. Assets are securely registered via the Resource Manager (`at="Foot"`) to prevent DOM blocking.

```html
@using System.Text.Json
@inject Microsoft.AspNetCore.Mvc.Localization.IViewLocalizer T

@{
    // Secure localization for JavaScript via C# Dictionary to prevent syntax breaks
    var translations = new Dictionary<string, string>
    {
        ["selectActivity"] = T["Select activity and model..."].Value,
        ["searchModel"] = T["Search model..."].Value
    };
    var translationsJson = JsonSerializer.Serialize(translations);
}

<!-- Optimized loading of Syncfusion assets via OC Resource Manager -->
<style asp-name="syncfusion-material"></style>
<script asp-name="syncfusion-js" at="Foot"></script>

<script at="Foot">
    // Safely passing backend translations to a global JavaScript object
    window.aiTranslations = @Html.Raw(translationsJson);
    
    // Initializing Syncfusion DropDownList with server-side data mapping
    document.addEventListener('DOMContentLoaded', function () {
        const modelData = new ej.data.DataManager({
            url: '/api/content-ai/ai-models/available',
            adaptor: new ej.data.WebApiAdaptor(),
            crossDomain: false
        });

        const dropDownListObj = new ej.dropdowns.DropDownList({
            dataSource: modelData,
            fields: { text: 'name', value: 'id', groupBy: 'capability' },
            placeholder: window.aiTranslations.selectActivity,
            filterBarPlaceholder: window.aiTranslations.searchModel,
            allowFiltering: true
        });
        dropDownListObj.appendTo('#model-name-dropdown');
    });
</script>
```
## 2. Clean Google Gemini API Integration (.NET/C#)
A snippet from GeminiClient.cs showcasing asynchronous API calls, Dependency Injection, and reading dynamic module configuration securely from Orchard Core's ISiteService.  
```csharp
/// <summary>
/// Asynchronously generates content using the Google Gemini API based on site settings.
/// </summary>
public async Task<string> GenerateContentAsync(string prompt, string systemInstruction = "", string? modelId = null)
{
    // Dynamically retrieve configuration from OC SiteSettings
    var siteSettings = await _siteService.GetSiteSettingsAsync();
    var settings = siteSettings.GetOrCreate<AiGeneratorSettings>();

    if (string.IsNullOrWhiteSpace(settings.GeminiApiKey)) 
        return string.Empty;

    var actualModel = string.IsNullOrWhiteSpace(modelId) ? "gemini-2.5-flash-lite" : modelId.Trim();
    var url = $"v1beta/models/{actualModel}:generateContent?key={settings.GeminiApiKey}";

    var requestBody = new
    {
        contents = new[] { new { parts = new[] { new { text = prompt } } } },
        systemInstruction = !string.IsNullOrWhiteSpace(systemInstruction) ? new { parts = new[] { new { text = systemInstruction } } } : null,
        generationConfig = new { responseMimeType = "application/json" }
    };

    using var content = new StringContent(JsonSerializer.Serialize(requestBody), Encoding.UTF8, "application/json");

    var response = await _httpClient.PostAsync(url, content);
    if (!response.IsSuccessStatusCode)
    {
        string errorDetails = await response.Content.ReadAsStringAsync();
        _logger.LogError("Google API Error. Status: {StatusCode}, Detail: {ErrorDetails}", response.StatusCode, errorDetails);
        throw new Exception($"Gemini API rejected the request ({response.StatusCode}).");
    }

    var jsonResponse = await response.Content.ReadFromJsonAsync<JsonElement>();
    return jsonResponse.GetProperty("candidates")[0]
                       .GetProperty("content")
                       .GetProperty("parts")[0]
                       .GetProperty("text").GetString() ?? string.Empty;
}
```

## 3. Dynamic Liquid Template Compilation (C# Controller)
This advanced technique uses ILiquidTemplateManager to compile raw AI-generated JSON data using an admin-defined template from the OC Templates module. It handles safe JsonElement to FluidValue conversion to ensure compatibility with the Orchard Core Liquid engine.  
```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Create(string targetContentType, string articleDataJson, string pipelineTemplate)
{
    // 1. Initialize a new ContentItem via IContentManager
    var contentItem = await _contentManager.NewAsync(targetContentType);

    // 2. Liquid Template Compilation
    if (!string.IsNullOrWhiteSpace(pipelineTemplate) && !string.IsNullOrWhiteSpace(articleDataJson))
    {
        using (var jsonDoc = JsonDocument.Parse(articleDataJson))
        {
            // Convert raw JSON to a Fluid-compatible dictionary structure
            var articleData = ConvertToFluidFriendlyObject(jsonDoc.RootElement);

            var properties = new Dictionary<string, FluidValue>
            {
                ["article"] = FluidValue.Create(articleData, new TemplateOptions()),
                ["ContentItem"] = FluidValue.Create(contentItem, new TemplateOptions())
            };

            // Render the Liquid template dynamically
            string finalHtml = await _liquidTemplateManager.RenderStringAsync(
                pipelineTemplate,
                HtmlEncoder.Default,
                null, 
                properties);

            // Inject the generated HTML into the native OC HtmlBodyPart
            contentItem.Content["HtmlBodyPart"] = new { Html = finalHtml };
        }
    }

    // 3. Persist the draft safely using OC ContentManager
    await _contentManager.CreateAsync(contentItem, VersionOptions.Draft);
    
    return RedirectToAction("Edit", "Admin", new { area = "OrchardCore.Contents", contentItemId = contentItem.ContentItemId });
}
```

## 4. Fluent API Schema Definition (Code-First Migrations)
Demonstrates the use of Code-First migrations and the IContentDefinitionManager to build robust ContentType and ContentPart definitions programmatically, ensuring clean deployments without relying on manual admin configuration.
```csharp
public async Task<int> UpdateFrom1Async()
{
    // 1. Define AI Agent Part with UI fields, hints, and validation
    await _contentDefinitionManager.AlterPartDefinitionAsync("AiAgentPart", builder => builder
        .Attachable()
        .WithDescription("Defines configuration fields for AI agents (models, instructions, contexts).")
        .WithField("ModelName", field => field
            .OfType("TextField")
            .WithDisplayName("Model")
            .WithDescription("Specific model ID from the AI provider.")
            .WithSettings(new TextFieldSettings { Required = true, Hint = "Default is gemini-1.5-flash" }))
        .WithField("SystemInstruction", field => field
            .OfType("TextField")
            .WithDisplayName("System Instructions for the Agent")
            .WithEditor("TextArea")));

    // 2. Bind the Part and system TitlePart into a complete ContentType
    await _contentDefinitionManager.AlterTypeDefinitionAsync("AiAgent", type => type
        .WithDisplayName("AI Agent")
        .WithDescription("Content type representing a specific AI persona/role.")
        .Draftable()
        .Versionable()
        .WithPart("TitlePart", part => part.WithPosition("1"))
        .WithPart("AiAgentPart", part => part.WithPosition("2")));

    return 2;
}
```
