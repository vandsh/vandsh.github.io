---
layout: post
title: "Unit Testing in Testing Sitecore"
category: Sitecore
tags: Sitecore Unit-Testing
published: true
---

Unit testing is a highly debated topic especially around the benefits versus the overhead costs and maintainence of said tests.  I am just going to toss out my 2 cents...

## Prelude
Just some background to give some context on my thoughts, feel free to skip to the next section if you don't actually care.

_Cues flashback animation_

![alt text](/assets/gifs/flashback.gif "A long time ago")

In a previous "enterprisey" life, I worked on a team that was diehards on code quality since everything we wrote was custom and ours.  We used the following flow for _every_ business ask:

- All new features defined as `User Stories`
- `User Stories` translated to tests with [SpecFlow](http://specflow.org/)
- [nUnit](http://nunit.org/) used to execute test on the underlying business logic within the codebase
- [Moq](https://www.nuget.org/packages/moq/) used to mock services and repositories (proper dependency injection)
- [Selenium](http://www.seleniumhq.org/) used to smoke test the front end portions of the tests

_Cues flashforward animation to current time_

![alt text](/assets/gifs/forward.gif "Back to today...")
 
After the enterprise world, I was thrust into the wonderful world of content management systems and consulting. A world where unit tests were put under a microscope for the following reasons (not only by clients but by myself as well):

- Cost of maintaining code that "just tests" is a lot to assume
- Do we really need to test a commercial/community tested product _(and this is the hardest one to argue against)_
- We have a tight budget and just want deliverables

So given the above, it's been hard to sell the idea of unit tests until a recent Sitecore project forced me to reassess these notions.

## As part of an excercise, I completed the following:

### Identify areas not considered out-of-box Sitecore
Focus on things that were clearly custom written by I or one of the other team members.  The key here was to assume that all pieces of Sitecore were either already tested or commercially supported in the even there _was_ a bug. Look soley at what we wrote that was not just putting fields in a rendering.

For example, `Search`. Yes, Sitecore has search mechanisms, but when it comes to actual filtering and faceting logic of the search itself, you end up writing most of it.
```csharp
//Super basic service that contains opening a 
//ProviderSearchContext and closing all within the service
//Most likely some PredicateBuilder logic to filter by template etc.
var _searchService = new SearchService();
_searchService.SiteSearch(query, pageSize, pageNumber);
```

### Lightly refactor things
Clean up services and business logic to have their dependencies injected, code to interfaces:

```csharp
//Abstract and inject the dependencies so we can mock all the underlying index and 
//.Queryable pieces and focus on the logic in the service  
public SearchService(IProviderSearchContext searchContext, 
                     ISearchRepository searchRepository)
{
    _searchContext = searchContext;
    _searchRepository = searchRepository;
}
```

Avoid static/extension methods as much as physically possible (this proved to be the most challenging).
Convert most if not all of the extension and static methods to be "helper" type classes that contain proper instances (more easily mockable):

```csharp
((string)searchTagValue).GetTagItem();

//converted to

var _tagHelper = new TagHelper(); //injected via constructor of course...
...
_tagHelper.GetTagItem(searchTagValue);
```
Now that we converted it, and inside the method it was doing a `Sitecore.Database.GetItem` we can now mock it effectively eliminating the Sitecore call all together (or mock pieces of it using `Sitecore.FakeDb`).
Converting these types of classes took a while since they both had to be refactored and injected (and since it was an extension method, it was used _a lot_ at will). This did also include wrapping some `ContentSearch` extensions like `GetResults` and `Where` so they could be mocked against an `IEnumerable` instead of a proper `Lucene\Solr` index):

```csharp
public class FakeSearchExtensions : ISearchExtensions
{
    public SearchResults<TSource> GetResults<TSource>(IQueryable<TSource> source)
    {
        //Lazy test, simply casting since where executed predicate
	Trace.Write("FakeSearchExtensions.GetResults");
	var sourceList = source.AsEnumerable()
            .Select(x => new SearchHit<TSource>(1, x));
	var searchResults = new SearchResults<TSource>(sourceList, 
            sourceList.Count(), null);
	return searchResults;
    }

    public IQueryable<TSource> Where<TSource>(IQueryable<TSource> source, 
        Expression<Func<TSource, bool>> predicate)
    {
	//Lazy test, not deferring the filter, execute immediately
	Trace.Write("FakeSearchExtensions.Where");
	var compiledQuery = predicate.Compile();
	var queryExecuted = source.AsEnumerable().Where(compiledQuery).ToList();
	return queryExecuted.AsQueryable();
    }
}

///Elsewhere in solution...

    var queryRunner = _searchExtensions.Where(_searchContextQueryable, baseQuery);
    var results = _searchExtensions.GetResults(queryRunner);
```

As challenging as this was, it let me mock all the underlying search mechanisms and focus soley on the desired logic (faceting, filtering, and/ors, text searches etc.)

### Use what you know
Moq, nUnit and Selenium were the weapons of choice based on familiarity, though there are many other standard tools I could have used.

```csharp
[Test]
[Category("UnitTest")]
public void ExecuteSearchQuery()
{
	//the search context most often will just be the sitecore search context 
        //but could be outside of sitecore
	var mockSearchProvider = new Mock<IProviderSearchContext>();
	
	//the search repository consumes the provider which most often 
        //will be sitecore but could be another provider and another mechanism 
	var mockSearchRepository = new Mock<ISearchRepository>();

	//the search service instance we want to mock
	var searchService = new SearchService(mockSearchProvider.Object, 
                                               mockSearchRepository.Object);

	//mock the repository

	var mockResultsList = new List<SearchHit<SearchResult>>();
	mockResultsList.Add(new SearchHit<SearchResult>(1, new SearchResult()
	{
		Content = "test",
		CreatedBy = "admin",
		CreatedDate = DateTime.MinValue,
		DatabaseName = "web",
		Datasource = null,
		ExcludeFromSearch = false,
		FullLinkToItem = "http://linktoitem",
		IsPage = true
	}));

        var mockSearchResults = new SearchResults<SearchResult>(mockResultsList, 1);

	mockSearchRepository
		.Setup(ss => ss.Search(mockSearchProvider.Object, 
                                        It.IsAny<string>(), 
                                        It.IsAny<int>(), 
                                        It.IsAny<int>()))
		.Returns(mockSearchResults);

	//test the rest of the search using the mocked provider
	var searchResultsDto = searchService.SiteSearch("test", 1, 10);

	//assert
	searchResultsDto.Hits.Count().Should().Be(1);
} 
```

### When testing pieces of Sitecore is unavoidable
Limit the test surface with Sitecore itself as much as possible but when it isn't possible to avoid, utilize things like [Sitecore.FakeDB](https://www.nuget.org/packages/Sitecore.FakeDb/)


```csharp
//setup
var testTag = new DbItem("test1", ID.NewID, ITagConstant.TemplateId);
var _db = new Db(){testTag};

//test the innerds of the tagHelper class that actually calls .GetItem...
var _mockSitecoreContext = new Mock<ISitecoreContext>();
_mockSitecoreContext.Setup(sc => sc.Database).Returns(_db.Database);
var tagHelper = new TagHelper(_mockSitecoreContext.Object);
tagHelper.GetTagItem("test1");

//teardown
_db.Dispose();
```

## Findings
After this enlightening exercise I concluded a few things:
- No matter how well you know the code, when you start breaking things down into direct asks, you can find flaws in your own logic (which was surely the case here)
- If I were to have broken up things into unit tests before ever writing a line of implementation code, I may have wrote things entirely different and would have been able to hand it off to _any_ other developer to tackle.
- The compromise in unit testing a commercial CMS on _only what you write_ and trying to steer clear of _what's already provided_ is certainly attainable.

I hope that some part of this exercise can help you as much as it has me in understanding the benefits of unit testing within a CMS and that there is a always going to be a fine line of "too far" and "just enough".
