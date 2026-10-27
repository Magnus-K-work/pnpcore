# Plan: Add `PageNames` filter to PnP.Core.Provisioning page extraction

Repo: `C:\Users\magka24\source\repos\pnpcore` (upstream: https://github.com/pnp/pnpcore, target branch `dev`)

## Background
- Page extraction only supports home page or all pages (`ExtractPagesConfiguration.IncludeAllClientSidePages`).
- Extracting a single page means extracting all pages and filtering afterwards (`template.ClientSidePages.RemoveAll(...)`).
- With `ExtractConfiguration.PersistAssetFiles = true` (+ `FileConnector`), assets of ALL pages are downloaded and end up in `template.Files`, even if pages are removed afterwards.
- PnP.Framework had a public `ClientSidePageContentsHelper` for single page extraction; in pnpcore it is `internal sealed`, as are `PageToExport` and its properties.

## Verified code facts
- `src/sdk/PnP.Core.Provisioning/Model/Configuration/Pages/ExtractPagesConfiguration.cs`: only `ExcludeAuthorInformation` and `IncludeAllClientSidePages`.
- `src/sdk/PnP.Core.Provisioning/ObjectHandlers/ObjectClientSidePageContents.cs`:
  - `ExtractObjectsAsync` queries all `.aspx` items, builds `List<PageToExport>` via `SurveyPages`, then `LinkTranslationsToTheirSources`.
  - Loop at ~lines 124-142: `if (includeAllPages || page.IsHomePage)` -> `helper.ExtractClientSidePageAsync(...)`.
  - `PageName` = page url relative to the pages library (includes subfolder, e.g. `Folder/Page.aspx`).
- `ClientSidePageContentsHelper.ExtractClientSidePageAsync(PnPContext, ProvisioningTemplate, ExtractConfiguration, PageToExport, ProvisioningMessagesDelegate)` extracts one page; asset persistence happens inside it.

## Steps

### 1. Extend configuration
File: `src/sdk/PnP.Core.Provisioning/Model/Configuration/Pages/ExtractPagesConfiguration.cs`
```csharp
using System.Collections.Generic;
using System.Text.Json.Serialization;

namespace PnP.Core.Provisioning.Model.Configuration.Pages
{
	public class ExtractPagesConfiguration
	{
		[JsonPropertyName("excludeAuthorInformation")]
		public bool ExcludeAuthorInformation { get; set; }

		[JsonPropertyName("includeAllClientSidePages")]
		public bool IncludeAllClientSidePages { get; set; }

		/// <summary>
		/// Names of the pages to extract, relative to the pages library (e.g. <c>News.aspx</c> or
		/// <c>Folder/News.aspx</c>). When set, only these pages (and their translations) are extracted
		/// and <see cref="IncludeAllClientSidePages"/> is ignored.
		/// </summary>
		[JsonPropertyName("pageNames")]
		public List<string> PageNames { get; set; } = new List<string>();
	}
}
```

### 2. Filter in the extractor
File: `src/sdk/PnP.Core.Provisioning/ObjectHandlers/ObjectClientSidePageContents.cs`

Replace the selection/loop block (~lines 124-142) with:
```csharp
bool includeAllPages = configuration?.Pages?.IncludeAllClientSidePages ?? false;
var requestedPages = new HashSet<string>(
	configuration?.Pages?.PageNames ?? Enumerable.Empty<string>(),
	StringComparer.OrdinalIgnoreCase);

List<PageToExport> selectedPages = pagesToExport
	.Where(p => IsSelected(p, requestedPages, includeAllPages))
	.OrderBy(p => p.IsTranslation)
	.ToList();

var helper = new ClientSidePageContentsHelper();

await context.Web.LoadAsync(w => w.ServerRelativeUrl).ConfigureAwait(false);

int currentPageIndex = 1;
foreach (PageToExport page in selectedPages)
{
	WriteSubProgress("ClientSidePage",
		!string.IsNullOrWhiteSpace(page.PageName) ? page.PageName : page.PageUrl,
		currentPageIndex++, selectedPages.Count);

	await helper.ExtractClientSidePageAsync(context, template, configuration, page, MessagesDelegate).ConfigureAwait(false);
}

return template;
```

Add to the same class:
```csharp
/// <summary>
/// Decides whether a surveyed page is extracted. An explicit page list wins over
/// <c>IncludeAllClientSidePages</c>; translations follow their source page.
/// </summary>
private static bool IsSelected(PageToExport page, HashSet<string> requestedPages, bool includeAllPages)
{
	if (requestedPages.Count > 0)
	{
		return requestedPages.Contains(page.PageName)
			|| (page.IsTranslation && !string.IsNullOrEmpty(page.SourcePageName)
				&& requestedPages.Contains(page.SourcePageName));
	}

	return includeAllPages || page.IsHomePage;
}
```

### 3. Verify (open items)
- [ ] Read `LinkTranslationsToTheirSources` and confirm `SourcePageName` uses the same format as `PageName`.
- [ ] Check whether `ProvisioningTemplateCreationInformation` / `ExtractConfiguration.FromCreationInformation` / `ToCreationInformation` need a mapping for `PageNames` (backwards compat path).
- [ ] Check for a JSON schema or docs describing the extract configuration; add `pageNames`.
- [ ] Check `PnP.Core.Test` for provisioning extraction tests (and how mocked/recorded tests work); add tests:
  - `PageNames` empty -> unchanged behaviour (home page / all pages).
  - `PageNames` with one page -> only that page in `template.ClientSidePages`.
  - `PageNames` + `PersistAssetFiles` -> only that page's assets in `template.Files`.
  - Page in subfolder (`Folder/Page.aspx`).
  - Translations of a requested page are included.
- [ ] Read CONTRIBUTING guidelines; branch from `dev`.

### 4. Optional (separate PR)
- Narrow the CAML query by `FileLeafRef` when `PageNames` is set, to avoid loading all `.aspx` items.

### 5. Validate with a demo project
- Add a local (uncommitted) console project referencing `PnP.Core.Provisioning.csproj`.
- Reuse POC setup: certificate auth, `FileSystemConnector` on a per-run temp folder, `PersistAssetFiles = true`, set `template.Connector` before `ApplyTemplateAsync`.
- Call with `Pages = { PageNames = { "Health-News.aspx" } }` and confirm only that page and its assets are extracted.

## PR description draft
**Title:** Allow extracting specific client side pages via `ExtractPagesConfiguration.PageNames`

**Problem**
Extraction currently supports either the home page only or all pages (`IncludeAllClientSidePages`). Extracting a single page requires extracting all pages and filtering afterwards; with `PersistAssetFiles = true` this also downloads the assets of every page. PnP.Framework allowed single page extraction via the public `ClientSidePageContentsHelper`, which is internal in PnP.Core.Provisioning.

**Change**
- Added `PageNames` to `ExtractPagesConfiguration` (JSON: `pageNames`).
- When non-empty, only matching pages (and their translations) are extracted; `IncludeAllClientSidePages` is ignored.
- Unchanged behaviour when empty.
- Progress reporting counts selected pages only.

**Usage**
```csharp
var template = await context.GetProvisioningManager().GetTemplateAsync(new ExtractConfiguration
{
	Handlers = { ConfigurationHandler.Pages },
	Pages = { PageNames = { "Health-News.aspx" } }
});
```
