# Full Text Search
FullTextSearch works by extending the default `ExternalIndex` in Umbraco with its own fields for full text content. It renders nodes using Umbracos own `RenderTemplate` method, and indexes the content.

The rendered content of nodes is cached in a seperate database table, so rendering can be skipped when reindexing Examine. FullTextSearch listens to events from the `ContentCacheRefresher`, and updates its own cache upon the `CacheUpdated` event.

## Installation
Install using the nuget package.
This package requires Umbraco 9.0 as a minimum version.

## Configuration
FullTextSearch is configured using configuration in your Startup.cs file. You can use the default configuration without doing anything, but if you want to do your own configuration, you can add it to your ConfigureServices method in Startup.cs like this:

```cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddUmbraco(_env, _config)
        .AddBackOffice()
        .AddWebsite()
        .AddComposers()
        .Build();

    // using Our.Umbraco.FullTextSearch.Options
    services.Configure<FullTextSearchOptions>(o =>
    {
        o.Enabled = false; // set your own config here.
    });
}
```

FullTextSearch config comes with a [default config, you can see it here](/src/Our.Umbraco.FullTextSearch/Options/FullTextSearchOptions.cs)

Here is an overview of the different config settings you can add.

### Indexing
Indexing the full text content is by default enabled, but you can disable it by setting `Enabled` to false.

#### Default Title Field
The `DefaultTitleField` node contains the name of the field containing the title of the page in the index. The default value is `nodeName`. You can also override this when searching.

#### Disallowed Aliases
By default, all nodes with a template will be cached and indexed. You can control which nodes are being indexed, by adding the aliases of their Content Types, or the alias of a checkbox property on the nodes to exclude them from this.

The disallowed property aliases can be used to create your own "umbracoNaviHide" for FullTextSearch. Add a boolean property to your document type, and reference its alias in this setting. If the node has that property set to true, it will be excluded from full text indexing.

Adding to this config on a site already indexed doesn't clean the index. You have to do this manually - but the search will exclude nodes of the disallowed types for you.

By default, no content types or property aliases is disallowed.

#### Rendering
When rendering, FullTextSearch adds the value of `IndexingActiveKey` as a key in HttpContext.Items[], so you can use that to send different content to the indexer. The default value is FullTextIndexingActive. You can also use the `IsIndexingActive` helper method, in your views, to determine whether or not indexing is active. You can use this to exclude parts of the views from the content being indexed.

##### XPaths to remove
In addition to the IndexingActiveKey, you can also add specific Xpaths to remove from the indexed content. Using this, you can ie. remove scripts (`//script`) or the head area ('//head') of the page.

By default no XPaths are removed.

#### Examine Field Names
Full Text Search works by extending the default `ExternalIndex` in Umbraco. Fields containing the full text content, and a searchable path to the node is added. By default these fields are called FullTextContent, and FullTextPath, but if you need them to be something else (like, if you had Umbraco properties using those aliases), you can change it in this config section.

## Searching

### The clean way
The preferred way of searching is by using the `ISearchService` interface. Below is the simplest possible example of a controller for a search node.
```
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.ViewEngines;
using Microsoft.Extensions.Logging;
using Our.Umbraco.FullTextSearch.Interfaces;
using Our.Umbraco.FullTextSearch.Models;
using Umbraco.Cms.Core.Web;
using Umbraco.Cms.Web.Common.Controllers;
using Umbraco.Extensions;

namespace MyProject.Controllers
{
    public class SearchController : RenderController
    {
        private readonly ISearchService _searchService;

        public SearchController(ISearchService searchService, ILogger<RenderController> logger, ICompositeViewEngine compositeViewEngine, IUmbracoContextAccessor umbracoContextAccessor) : base(logger, compositeViewEngine, umbracoContextAccessor)
        {
            _searchService = searchService;
        }

        public override IActionResult Index()
        {
            if (Request.Query.ContainsKey("q"))
            {
                int.TryParse(Request.Query["p"].ToString(), out int currentPage);
                currentPage = currentPage < 1 ? 1 : currentPage;

                var search = new Search(Request.Query["q"].ToString())
                    .SetCulture(CurrentPage.GetCultureFromDomains().ToLower());

                ViewBag.FullTextSearchResult = _searchService.Search(search, currentPage);
            }
            return CurrentTemplate(CurrentPage);
        }
    }
}
```
In this example I add `ISearchService` as a dependency for the controller. `ISearchService` performs the searching, and takes a `Search` object, which can be configured in numerous ways. The result is then added to the `ViewBag`. Notice that you need to set the culture of the search in order to get results from the right fields. Also notice the `Request.Query["p"]` and `Request.Query["q"]` variables used for paging and getting search terms.

My preferred way of searching looks like this:

```
using Our.Umbraco.FullTextSearch.Interfaces;

namespace Umbraco.Cms.Web.Common.PublishedModels
{
    public partial class Search
    {
        public IFullTextSearchResult FullTextSearchResult { get; set; }
    }
}
```
First I extend the model of my search page from ModelsBuilder, by adding a partial class, and adding the `IFullTextSearchResult` property.


```
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.ViewEngines;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using Our.Umbraco.FullTextSearch.Interfaces;
using Our.Umbraco.FullTextSearch.Models;
using Our.Umbraco.FullTextSearch.Options;
using Umbraco.Cms.Core.Web;
using Umbraco.Cms.Web.Common.Controllers;
using Umbraco.Extensions;
using ContentModels = Umbraco.Cms.Web.Common.PublishedModels;

namespace MyProject.Controllers
{
    public class SearchController : RenderController
    {
        private readonly ISearchService _searchService;
        private readonly FullTextSearchOptions _options;

        public SearchController(ISearchService searchService, IOptions<FullTextSearchOptions> options, ILogger<RenderController> logger, ICompositeViewEngine compositeViewEngine, IUmbracoContextAccessor umbracoContextAccessor) : base(logger, compositeViewEngine, umbracoContextAccessor)
        {
            _searchService = searchService;
            _options = options.Value;
        }

        public override IActionResult Index()
        {
            var searchModel = CurrentPage as ContentModels.Search;

            if (Request.Query.ContainsKey("q"))
            {
                int.TryParse(Request.Query["p"].ToString(), out int currentPage);
                currentPage = currentPage < 1 ? 1 : currentPage;

                var search = new Search(Request.Query["q"].ToString())
                    .EnableHighlighting()
                    .AddTitleProperty("metaTitle")
                    .AddTitleProperty("nodeName")
                    .AddSummaryProperty("metaDescription")
                    .AddSummaryProperty(_options.FullTextContentField)
                    .SetSummaryLength(160)
                    .SetPageLength(10)
                    .SetCulture(CurrentPage.GetCultureFromDomains().ToLower());

                searchModel.FullTextSearchResult = _searchService.Search(search, currentPage);
            }
            else
            {
                searchModel.FullTextSearchResult = null;
            }
            return CurrentTemplate(searchModel);
        }
    }
}

```
Here I also include `IOptions<FullTextSearchOptions>` as a dependency, enabling me to get different keys from the config in my configuration. I then add a new property (`FullTextSearchResult`) to my generated model from ModelsBuilder, where I can put my search results. As you can see, the `new Search()` part is now configured in more detail. See what you can configure further below.

On this search, I enable highlighting of text in the output by using `EnableHightling()`, I then configure it to use `metaTitle` and `nodeName` as fields for getting the title of each page (that way `nodeName` is used, if `metaTitle` is empty or doesn't exist). I configure it to use `metaDescription` or the default text field for the summary. And then I set the summary length to be 160 characters (the default is 300), and the pagelength to be 10 results per page (the default is 0, meaning no pagination will occur). Lastly I set the culture, to make FullTextSearch look for results with the same culture as the request.

### The easy way
By injecting the FullTextSearch helper into your view, you can search directly from your Razor view. The simplest possible way of doing that, is by simple adding the following to your view:

```
@inject Our.Umbraco.FullTextSearch.Helpers.FullTextSearchHelper FullTextSearchHelper
@{
    var searchResult = FullTextSearchHelper.Search("putYourSearchTermHere");
}
```

The Search helper method takes either a string searchTerm, and an optional culture string, or a `Search` object, like demonstrated in the controllers above. In addition to that, you can specify which page of the search results you want to show. By default, the first page will be shown, but if you wan't paging you need to implement that by your self. Feel free to take inspiration from the controllers.


## Rendering the search results
When rendering you work with the `IFullTextSearchResult` value, that you either got using a controller or the helper method. The code sample below shows how to use it from the ModelsBuilder extension, but `Model.FullTextSearchResult` could also have been `ViewBag.FullTextSearchResult` or simply `searchResult` following the examples above.

```
@if (Model.FullTextSearchResult != null)
{
    <ul>
        @foreach (var result in Model.FullTextSearchResult.Results)
        {
            <li>
                <a href="@result.Content.Url()">@result.Title</a>
                <p>
                    @result.Summary
                </p>
                <small>
                    Url: @result.Content.Url()<br />
                    Id: @result.Id<br />
                    Last updated: @(result.Content.UpdateDate)<br />
                    Score: @result.Score
                </small>
            </li>
        }
    </ul>

    <div>Total results: @Model.FullTextSearchResult.TotalResults</div>

    if (Model.FullTextSearchResult.CurrentPage > 1)
    {
        <a href="?q=@Context.Request.Query["q"].ToString()&p=@(Model.FullTextSearchResult.CurrentPage-1)">Previous page</a>
    }

    if (Model.FullTextSearchResult.CurrentPage < Model.FullTextSearchResult.TotalPages)
    {
        <a href="?q=@Context.Request.Query["q"].ToString()&p=@(Model.FullTextSearchResult.CurrentPage+1)">Next page</a>
    }
}

```

Model.FullTextSearchResults now contains the result object (`IFullTextSearchResult`), in which the following properties is available:

`long TotalPages`
The total number of search result pages.

`long CurrentPage`
The current search result page being viewed.

`long TotalResults`
The total number of search results.

`IEnumerable<ISearchResultItem> Results`
A list of results on the current page.

Each result is an `ISearchResultItem` with the following properties:

`string Id`
The id of the node.

`double Score`
The score given by Lucene to the result. The higher the better.

`IReadOnlyDictionary<string, string> Fields`
The fields on the item in Examine

`string Title`
The title generated by FullTextSearch according to your title properties.

`IHtmlString Summary`
The summary generated by FullTextSearch according to your summary properties and summary length.

`IPublishedContent Content`
The IPublishedContent for the search result.

## Configuring the search
The search service takes a `Search` object, which can be configured fluently in numerous ways. You create one by writing `new Search(string searchTerm)`, and from there you can add the following configuration:

`AddTitleProperties(string[] aliases)` or `AddTitleProperty(string alias)`
Adds field names to use for title properties. Note, that this overrides the config setting, so you need to add all wanted fields for titles here.

`RemoveTitleProperties(string[] aliases)` or `RemoveTitleProperty(string alias)`
This removes field names to use for title properties. Note that the default title properties from config is only used if no other title properties are set in the search. So you don't have to remove the default and add another if you want to swap.

`AddBodyProperties(string[] aliases)` or `AddBodyProperty(string alias)`
Adds field names to use for body properties. Note, that this overrides the config setting, so you need to add all wanted fields for bodytext here.

`RemoveBodyProperties(string[] aliases)` or `RemoveBodyProperty(string alias)`
This removes field names to use for bodyproperties. Note that the default body properties from config is only used if no other body properties are set in the search. So you don't have to remove the default and add another if you want to swap.

`AddSummaryProperties(string[] aliases)` or `AddSummaryProperty(string alias)`
Adds field names to use for summary properties. Note, that if you don't specify any summary properties, the body properties will be used instead.

`RemoveSummaryProperties(string[] aliases)` or `RemoveSummaryProperty(string alias)`
This removes field names to use for summary properties.

`AddRootNodeIds(int[] ids)` or `AddRootNodeId(int id)`
With this setting, you can limit search results to be descendants of the input ids. This way you can make the search work on specific parts of your site.

`RemoveRootNodeIds(int ids)` or `RemoveRootNodeId(int id)`
Removes already set root node ids from the search object.

`SetSummaryLength(int length)`
Sets the summary length in the results. The default is `300` characters.

`SetFuzzyness(double fuzzyness)`
Fuzzyness is used to match your search term with similar words. This method sets the fuzzyness parameter of the search. The default is `0.8`.

`EnableWildcards()` or `DisableWildcards()`
These enables or disables use of wildcards in the search terms. Wildcard characters are added automatically to each of the terms. Wildcards is disabled by default

`EnableHighlighting()` or `DisableHighlighting()`
These enables or disables search term highlighting in the search results. If enabled, search terms in the summary will be highlighted by adding `<b>` tags around them. Highlighting is disabled by default.

`SetPageLength(int length)`
This sets the desired page length of your search results. If you set it to 0, you will get all search results on the first page. The default value is 0.

`SetCulture(string culture)`
This is used to define which culture to search in. You should probably always set this, but it might work without it, in invariant sites.

`AddAllowedContentTypes(string[] aliases)` or `AddAllowedContentType(string alias)`
Adds Content Type aliases to the list of allowed Content Types, in order to limit the search to nodes of the specified Content Type Aliases. By default, nodes of all Content Types (except the ones that are disallowed) will be searched.

`RemoveAllowedContentTypes(string[] aliases)` or `RemoveAllowedContentType(string alias)`
Removes Content Type aliases from the list of allowed Content Types.
