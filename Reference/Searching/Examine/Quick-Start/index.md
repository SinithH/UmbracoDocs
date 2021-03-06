---
versionFrom: 8.0.0
---

# Quick start

_This guide will help you get setup quickly using Examine with minimal configuration options. Umbraco ships Examine with 3 indexes: internal, external, and members. The internal index should not be used for searching when returning results on a public website because it includes content that has not been published yet. Instead you can use the external index to get up and running._

## Performing a search

:::note
In the coming examples the Umbraco Starter Kit has been used, as it gives some content to be searched upon quickly. Some of the examples may require setting up templates, etc if you are following on your own site.
:::

The starter kit comes with some Templates, Document Types and content nodes set up already. We will use some of those to set up a search system.

We will make it possible to search on the _People_ page.

We will do this by adding a search bar on the template page. In the `people.cshtml` template, add this at the top, under the `<nav>` element:

```csharp
...
</nav>
-->
<div>
    <form action="@Model.Url" method="get">
        <input type="text" placeholder="Search" id="query" name="query" />
        <button>Search</button>
    </form>
</div>
<div class="employee-grid">
...
```
This will create an input field at the top of the page and make it redirect to the same page when submitted.
Now let's set up the search query using Examine.
Below the form, add this:
```csharp
<div>
    @{
        var searchTerm = string.Empty;
        searchTerm = string.IsNullOrEmpty(Request["query"])
            ? string.Empty
            : Request["query"];
        if (searchTerm == string.Empty)
        {
            <p>Enter search term</p>
        }
        else
        {
            //perform the search
        }
    }
</div>
```

Here we are getting the request query from the form. If the string is empty we ask them to submit a term, otherwise we will search.

To perform the search we will first need to get the index and the searcher. We will do that by adding the following to the `else` condition:

```csharp
else
{
    //perform the search
    //first we try to get the index, it is the ExternalIndex as we don't want to return unpublished things
    //it returns the index in the var index
    if(ExamineManager.Instance.TryGetIndex("ExternalIndex", out var index))
    {
        var searcher = index.GetSearcher();
        var results = searcher.CreateQuery("content").NodeTypeAlias("person").And().Field("nodeName", searchTerm).Execute();
        if (results.Any())
        {
            <ul>
                @foreach (var result in results)
                {
                    if (result.Id != null)
                    {
                        var node = Umbraco.Content(result.Id);
                        <li>
                            <a href="@node.Url">@node.Name</a>
                        </li>
                    }
                }
            </ul>
        }
        else
        {
            <p>No results found for query @searchTerm</p>
        }
    }
    return;
}
```
At this point we have chosen to use the External index and its searcher. The query is then built. I
n this case we are searching all `content` using the `person` Document Type, where the `nodeName` is equal to the search term that was typed in the input bar.
The final template looks like this:
```csharp
@inherits Umbraco.Web.Mvc.UmbracoViewPage<ContentModels.People>
@using Umbraco.Examine
@using ContentModels = Umbraco.Web.PublishedModels;
@{
    Layout = "master.cshtml";
}
@helper SocialLink(string content, string service)
{
    if (!string.IsNullOrEmpty(content))
    {
        <a class="employee-grid__item__contact-item" href="http://@(service).com/@content">@service</a>
    }
}
@Html.Partial("~/Views/Partials/SectionHeader.cshtml")
<section class="section">
    <div class="container">
        <!-- todo: implement department filter -->
        <!--
        <nav class="nav-bar nav-bar--center nav-bar--air-bottom">
            <a class="nav-link nav-link--black nav-link--active" href="">All</a>
            <a class="nav-link nav-link--black" href="">Marketing</a>
            <a class="nav-link nav-link--black" href="">Package People</a>
            <a class="nav-link nav-link--black" href="">Designers</a>
            <a class="nav-link nav-link--black" href="">Other</a>
        </nav>
        -->
        <div>
            <form action="@Model.Url" method="get">
                <input type="text" placeholder="Search" id="query" name="query" />
                <button>Search</button>
            </form>
        </div>

        <div>
            @{
                var searchTerm = string.Empty;
                searchTerm = string.IsNullOrEmpty(Request["query"])
                    ? string.Empty
                    : Request["query"];

                if (searchTerm == string.Empty)
                {
                    <p>Enter search term</p>
                }
                else
                {
                    if(ExamineManager.Instance.TryGetIndex("ExternalIndex", out var index))
                    {
                        var searcher = index.GetSearcher();
                        var results = searcher.CreateQuery("content").NodeTypeAlias("person").And().Field("nodeName", searchTerm).Execute();
                        if (results.Any())
                        {
                            <ul>
                                @foreach (var result in results)
                                {
                                    if (result.Id != null)
                                    {
                                        var node = Umbraco.Content(result.Id);
                                        <li>
                                            <a href="@node.Url">@node.Name</a>
                                        </li>
                                    }
                                }
                            </ul>
                        }
                        else
                        {
                            <p>No results found for query @searchTerm</p>
                        }
                    }
                    return;
                }
            }
        </div>
        <div class="employee-grid">
            @foreach (ContentModels.Person person in Model.Children)
            {
                <div class="employee-grid__item">
                    <div class="employee-grid__item__image" style="background-image: url('@person.Photo.Url')"></div>
                    <div class="employee-grid__item__details">
                        <h3 class="employee-grid__item__name">@person.Name</h3>
                        @if (!string.IsNullOrEmpty(person.Email))
                        {
                            <a href="mailto:@person.Email" class="employee-grid__item__email">@person.Email</a>
                        }
                        <div class="employee-grid__item__contact">
                            @SocialLink(person.FacebookUsername, "Facebook")
                            @SocialLink(person.TwitterUsername, "Twitter")
                            @SocialLink(person.LinkedInUsername, "LinkedIn")
                            @SocialLink(person.InstagramUsername, "Instagram")
                        </div>
                    </div>
                </div>
            }
        </div>
    </div>
</section>
```
## Different ways to query
Examine has a lot of different ways to query data. Building upon the example from before, here are a few other searches that can be done to get different data:
### Search through all nodes
Let's say you want to search through **all content nodes** by their **file names**. You could amend the query from before like this:
```csharp
var results = searcher.CreateQuery("content").Field("nodeName", searchTerm).Execute();
```
### Search using Lucene queries
To do the search like above, but only use Lucene to query, amend the query from before like this:
```csharp
var results = searcher.CreateQuery().NativeQuery("+__IndexType:content +nodeName:" + searchTerm).Execute();
```
### Search children of a specific node
To search through **all child nodes of a specific node** by their **bodyText property**, amend the query from before like this:
```csharp
var results = searcher.CreateQuery("content").ParentId(1105).Field("bodyText", searchTerm).Execute();
```
