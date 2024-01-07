# Power BI - API with Pagination Data Source

The example demonstrates how to obtain data from API endpoint with pagination for Power BI that supports scheduled refresh in Power BI Service. The **example.pbix** file contains implementation for [https://openlibrary.org](https://openlibrary.org) API that obtains **Software Architecture** books

## How it works
The data querying is defined in **GetDataFromApiWithPagination** function:

``` M
let
    Source = () =>
    let
        domain = "https://openlibrary.org",   
        pageSize = 100,   
        GetPageResponse = (pageNumber) =>
        let
            page_response = Json.Document(Web.Contents(domain, 
                    [
                        RelativePath = "search.json?q=software+architecture&page=" & Text.From(pageNumber) & "&limit=" & Text.From(pageSize)
                    ]
                )
            )
        in
            page_response,
        GetPageData = (page_response) => page_response[docs],
        first_page_response = GetPageResponse(1),
        total_pages = Number.RoundUp(first_page_response[numFound] / pageSize),
        pages = {1 .. total_pages},
        pagesData = List.Transform(pages, each GetPageData(GetPageResponse(_))),
        dataList = List.Union(pagesData),
        tableFromList = Table.FromList(dataList, Splitter.SplitByNothing(), null, null, ExtraValues.Error)
    in
        tableFromList
in
    Source
```

You can modify the provided function, specifically **domain** variable and value in **RelativePath** to consume data from different endpoints. You can further decorate the function with own parameters (i.e. API version, query parameters) and further reuse it to obtain data from multiple endpoints of same API.

## Example for API that requires authentication

Assuming you are required to use the following API endpoint as a data source:

![Data endpoint definition](./img/data.png)

The data will be returned as paginated list containing the meta-data about total available records (totalCount) and number of items in the page (pageSize). The URL parameter **page** defines the data batch to return. For example:

If **totalCount: 325**, **pageSize: 100**

Request to **/api/data/1** will return items from 1 to 100,  **/api/data/2** - items from 101 to 200, **/api/data/3** - items from 201 to 300, **/api/data/4** - items from 301 to 325

The endpoint also requires **Bearer** authentication parameter to be provided in the request header. The parameter must be obtained first from the following endpoint:

![Authentication endpoint definition](./img/auth.png)

The following modification of the function will do the job:

``` M
let
    Source = () =>
    let
        domain = "https://domain.com",
        login = "theLogin",
        password = "thePassword",
        auth_request_body = "{
        ""login"": """ & login & """,
        ""password"": """ & password & """
        }",
        auth_response = Json.Document(Web.Contents(domain, 
                [
                    RelativePath = "api/auth",
                    Headers=[#"Content-Type"="application/json"], 
                    Content=Text.ToBinary(auth_request_body)
                ]
            )
        ),
        access_token = auth_response[token],
        GetPageResponse = (pageNumber) =>
        let
            page_response = Json.Document(Web.Contents(domain, 
                    [
                        RelativePath = "api/data/" & Text.From(pageNumber),
                        Headers=[#"Content-Type"="application/json", #"Bearer"=access_token]
                    ]
                )
            )
        in
            page_response,
        GetPageData = (page_response) => page_response[items],
        first_page_response = GetPageResponse(1),
        total_pages = Number.RoundUp(first_page_response[totalCount] / first_page_response[pageSize]),
        pages = {1 .. total_pages},
        pagesData = List.Transform(pages, each GetPageData(GetPageResponse(_))),
        dataList = List.Union(pagesData),
        tableFromList = Table.FromList(dataList, Splitter.SplitByNothing(), null, null, ExtraValues.Error)
    in
        tableFromList
in
    Source
```

You can further decorate the function with own parameters (i.e. API version, query parameters, request body parameters) and further reuse it to obtain data from multiple endpoints of same API.

## References

Search for a M language functions definitions at [https://learn.microsoft.com/en-us/powerquery-m/](https://learn.microsoft.com/en-us/powerquery-m/) for additional information or specifications of any provided bit of code.