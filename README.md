# Medienreaktor.Meilisearch

Integrates Meilisearch into Neos. *This is Work-in-Progress!*

This package aims for simplicity and minimal dependencies. It might therefore not be as sophisticated and extensible as packages like [Flowpack.ElasticSearch.ContentRepositoryAdaptor](https://github.com/Flowpack/Flowpack.ElasticSearch.ContentRepositoryAdaptor), and to achieve this, some code parts had to be copied from these great packages (see Credits).

## ✨ Features

* ✅ Indexing the Neos Content Repository in Meilisearch
* ✅ Supports Content Dimensions for all node variants
* ✅ CLI commands for building and flushing the index
* ✅ Querying the index via Search-/Eel-Helpers and QueryBuilder
* ✅ Frontend search form, result rendering and pagination
* ✅ Faceting and snippet highlighting
* 🟠 Only indexing the Live-Workspace for now
* 🔴 No asset indexing (yet)
* 🔴 No autocomplete / autosuggest (this is currently not supported by Meilisearch)

## 🚀 Installation

Install via composer:

    composer require medienreaktor/meilisearch

## ⚙️ Configuration

Configure the Meilisearch client in your `Settings.yaml` and set the Endpoint and API Key:

```yaml
Medienreaktor:
  Meilisearch:
    client:
      endpoint: ''
      apiKey: ''
```

You can adjust all Meilisearch index settings to fit your needs (see [Meilisearch Documentation](https://www.meilisearch.com/docs/reference/api/settings)). All settings configured here will directly be passed to Meilisearch.

```yaml
Medienreaktor:
  Meilisearch:
    settings:
      displayedAttributes:
        - '*'
      searchableAttributes:
        - '__fulltext.text'
        - '__fulltext.h1'
        - '__fulltext.h2'
        - '__fulltext.h3'
        - '__fulltext.h4'
        - '__fulltext.h5'
        - '__fulltext.h6'
      filterableAttributes:
        - '__identifier'
        - '__dimensionshash'
        - '__path'
        - '__parentPath'
        - '__type'
        - '__typeAndSupertypes'
        - '_hidden'
        - '_hiddenBeforeDateTime'
        - '_hiddenAfterDateTime'
        - '_hiddenInIndex'
      sortableAttributes: []
      rankingRules:
        - 'words'
        - 'typo'
        - 'proximity'
        - 'attribute'
        - 'sort'
        - 'exactness'
      stopWords: []
      typoTolerance:
        enabled: true
        minWordSizeForTypos:
          oneTypo: 5
          twoTypos: 9
      faceting:
        maxValuesPerFacet: 100
```

Please do not remove, only extend, above `filterableAttributes`, as they are needed for base functionality to work.

After finishing or changing configuration, build the node index once via the CLI command `flow nodeindex:build`. 

Document NodeTypes should be configured as fulltext root (this comes by default for all `Neos.Neos:Document` subtypes):

```yaml
'Neos.Neos:Document':
  search:
    fulltext:
      isRoot: true
      enable: true
```

Properties of Content NodeTypes that should be included in fulltext search must also be configured appropriately:

```yaml
'Neos.NodeTypes:Text':
  search:
    fulltext:
      enable: true
  properties:
    text:
      search:
        fulltextExtractor: "${Indexing.extractHtmlTags(node.properties.text)}"
```

## 📖 Usage with Neos and Fusion

There is a built-in Content NodeType `Medienreaktor.Meilisearch:Search` for rendering the search form, results and pagination that may serve as a boilerplate for your projects. Just place it on your search page to start.

You can also use search queries, results and facets in your own Fusion components.

    prototype(Medienreaktor.Meilisearch:Search) < prototype(Neos.Neos:ContentComponent) {
        searchTerm = ${String.toString(request.arguments.search)}

        page = ${String.toInteger(request.arguments.page) || 1}
        hitsPerPage = 10

        searchQuery = ${this.searchTerm ? Search.query(site).fulltext(this.searchTerm).nodeType('Neos.Neos:Document') : null}
        searchQuery.@process {
            page = ${value.page(this.page)}
            hitsPerPage = ${value.hitsPerPage(this.hitsPerPage)}
        }

        facets = ${this.searchQuery.facets(['__type', '__parentPath'])}
        totalPages = ${this.searchQuery.totalPages()}
        totalHits = ${this.searchQuery.totalHits()}
    }

If you want facet distribution for certain node properties or search in them, make sure to add them to `filterableAttributes` and/or `searchableAttributes` in your `Settings.yaml`.

The search query builder supports the following features:

| Query feature                                | Description                                                |
|----------------------------------------------|------------------------------------------------------------|
| `query(context)`                             | Sets the starting point for this query, e.g. `query(site)` |
| `nodeType(nodeTypeName)`                     | Filters by the given NodeType, e.g. `nodeType('Neos.Neos:Document')` |
| `fulltext(searchTerm)`                       | Performs a fulltext search |
| `filter(filterString)`                       | Filters by given filter string, e.g. `filter('__typeAndSupertypes = "Neos.Neos:Document"')` (see [Meilisearch Documentation](https://www.meilisearch.com/docs/reference/api/search#filter))  |
| `exactMatch(propertyName, value)`            | Filters by a node property |
| `exactMatchMultiple(properties)`             | Filters by multiple node properties, e.g. `exactMatchMultiple(['author' => 'foo', 'date' => 'bar'])` |
| `sortAsc(propertyName)`                      | Sort ascending by property |
| `sortDesc(propertyName)`                     | Sort descending by property |
| `limit(value)`                               | Limit results, e.g. `limit(10)` |
| `from(value)`                                | Return results starting from, e.g. `from(10)` |
| `page(value)`                                | Return paged results for given page, e.g. `page(1)` |
| `hitsPerPage(value)`                         | Hits per page for paged results, e.g. `hitsPerPage(10)` |
| `count()`                                    | Get total results count for non-paged results |
| `totalHits()`                                | Get total hits for paged results |
| `totalPages()`                               | Get total pages for paged results |
| `facets(array)`                              | Return facet distribution for given facets, e.g. `facets(['__type', '__parentPath'])` |
| `highlight(properties, highlightTags)`       | Highlight search results for given properties, e.g. `highlight(['__fulltext.text'])`, highlighted with given tags (optional, default: `['<em'>, '</em>']`) |
| `crop(cropLength, cropMarker)`               | Sets the highlighting snippets length in words and the crop marker (optional, default: `'…'`) |
| `matchingStrategy(value)`                    | Sets the matching strategy `'last'` or `'all'`, (default: `'last'`)
| `execute()`                                  | Execute the query and return resulting nodes |
| `executeRaw()`                               | Execute the query and return raw Meilisearch result data, enriched with node data |

## ⚡ Usage with JavaScript / React / Vue

If you want to build your frontend with JavaScript, React or Vue, you can completely ignore above Neos and Fusion integration and use `instant-meilisearch`. 

Please mind two things:

1. Setup your filter to always include the following filter string:
`(__parentPath = "$nodePath" OR __path = "$nodePath") AND __dimensionshash = "$dimensionsHash"`
where `$nodePath` is the NodePath of your context node (e.g. site) and `$dimensionHash` is the MD5-hashed JSON-encoded context dimensions array.

2. There are no URLs (yet) in the Meilisearch index. Sorry! We would love to have this feature, probably by generating the URLs at indexing time – but it is just not implemented yet. (Contributions welcome!)

## 👩‍💻 Credits

This package is heavily inspired by and some smaller code parts are copied from:

+ [Sandstorm.LightweightElasticsearch](https://github.com/sandstorm/LightweightElasticsearch)
+ [Flowpack.ElasticSearch.ContentRepositoryAdaptor](https://github.com/Flowpack/Flowpack.ElasticSearch.ContentRepositoryAdaptor)
+ [Flowpack.SimpleSearch.ContentRepositoryAdaptor](https://github.com/Flowpack/Flowpack.SimpleSearch.ContentRepositoryAdaptor)
+ [Flowpack.SearchPlugin](https://github.com/Flowpack/Flowpack.SearchPlugin)

All credits go to the original authors of these packages.
