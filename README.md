# TestableHttpClient

Using HttpClient in code that is unit tested is seen as rather difficult, mainly because HttpClient itself can't be easily mocked and a lot of extension methods are used.  
TestableHttpClient aims to make it easy to stub different kind of responses and to assert the calls that are made via an HttpClient.

Some of the main features include:

- Easy response configuration which is inspired by how responses are made in ASP.NET Core.
- Support for timeouts, delays and sequenced responses, to be able to test retry mechanisms.
- Support for routing to get different responses.
- Asserting what requests were made.

## How to install

TestableHttpClient can be installed like any NuGet packages via the VisualStudio Package manager or via commandline:

```pwsh
dotnet add package TestableHttpClient
```

## Stub responses

One of the main features of TestableHttpClient is the ability to stub responses in order to avoid actual calls to the webserver.

The following example shows a basic example of this:

```csharp
TestableHttpMessageHandler handler = new();
handler.RespondWith(Responses.StatusCode(HttpStatusCode.NotFound));

GithubApiClient client = new(handler.CreateClient());

async Task act() => await client.GetOrganizationAsync("UnknownOrganization");
await Assert.ThrowsAsync<OrganisationNotFoundException>(act);
```

Configuring responses can be done by using the `Responses` class. Here we only use the `StatusCode` response, but there are several other response types. Probably the most used once are:

- `Json` to create a json serialized response
- `Text` to create a text based response

Both of these responses can be configure to set the encoding and the media-type and the status code, by default `HttpStatusCode.OK` is used.

Responses implement the `IRepsonse` class and if the default responses are not sufficient, you can provide your own.

## Testing resilience

One of the trickier things of testing network communiction is how to handle timeouts, delays and retries. TestableHttpClient provides three types of responses to help testing this:
- Timeout
- Delayed response
- Sequenced responses

### Timeout

Creating a unit tests that test how your code handles a timeout from the HttpClient is not straight forward. Normally you don't really want to change the settings of the HttpClient just for the test, but you also don't want to wait for the timeout to happen, since unittests should run fast.

`Responses.Timeout` is able to trigger the timeout for you without actually waiting for the timeout to happen.

```csharp
TestableHttpMessageHandler handler = new();
handler.RespondWith(Responses.Timeout());
GithubApiClient client = new(handler.CreateClient());

async Task act() => await client.GetOrganizationAsync("testablehttpclient");
await Assert.ThrowsAsync<GithubApiException>(act);
```

### Delayed responses

A delayed response delays returning the response by using a `Task.Delay` before returning the actual `HttpResponseMessage`.

Note that when the delay is greater than the configured timout on the HttpClient, a TaskCancelled exception will be thrown by the HttpClient instead.

```csharp
TestableHttpMessageHandler handler = new();
handler.RespondWith(Delayed(StatusCode(HttpStatusCode.NotFound), TimeSpan.FromSeconds(2)));
HttpClient httpClient = handler.CreateClient();
GithubApiClient client = new(httpClient);

async Task act() => await client.GetOrganizationAsync("UnknownOrganization");
await Assert.ThrowsAsync<OrganisationNotFoundException>(act);
```

### Sequenced responses

Using `Responses.Sequenced` multiple responses can be configured that will be used one after another. This could be useful for testing that a service is temporarily unavailable.

```csharp
TestableHttpMessageHandler handler = new();
handler.RespondWith(Sequenced(StatusCode(HttpStatusCode.ServiceUnavailable), succesResponse));
HttpClient httpClient = handler.CreateClient();
GithubApiClient client = new(httpClient);

Organization result = await client.GetOrganizationAsync("testablehttpclient");

Assert.Equal("testablehttpclient", result.Login);
Assert.Equal("TestableHttpClient", result.Name);
```

## Advanced response selection
Sometimes a simple response is not sufficient. This might because you need to make multiple requests, or want to only configure mulitple responses once and reuse the configuration over multiple integration tests.
In these cases you can either use `Responses.Route` or `Responses.SelectResponse`.

### Route based responses
`Reponses.Route` provides a simplified version of Endpoint routing in ASP.NET Core. It can select a response based on the request uri.

```csharp
TestableHttpMessageHandler handler = new();
handler.RespondWith(Route(builder =>
{
    builder.Map("/orgs/testablehttpclient", succesResponse);
    builder.MapFallBackResponse(StatusCode(HttpStatusCode.NotFound));
}));

HttpClient httpClient = handler.CreateClient();
GithubApiClient client = new(httpClient);

async Task act() => await client.GetOrganizationAsync("UnknownOrganization");
await Assert.ThrowsAsync<OrganisationNotFoundException>(act);

Organization result = await client.GetOrganizationAsync("testablehttpclient");

Assert.Equal("testablehttpclient", result.Login);
Assert.Equal("TestableHttpClient", result.Name);
```

### Custom response selection
When selecting a response based on the request uri is not sufficient and you want more control, you can use `Responses.SelectRespons` where you can provide a fucntion that, based on the `HttpRequestMessage` returns an `IResponse`.

## Custom responses
When the provided responses are not sufficient, you can implement `IResponse` to create your own responses.

In order to be able to create these responses consitently you can create extension methods on `IResponseExtensions`:

```csharp
class NoContent : IResponse
{
    public Task ExecuteAsync(HttpResponseContext context, CancellationToken _)
    {
        context.HttpResponseMessage.StatusCode = HttpStatusCode.NoContent;
    }
}

public static class Extensions
{
    public static IResponse NoContent(this IResponseExtensions _)
    {
        return new NoContent();
    }
}
```

In your test you can use:
```csharp
handler.RespondWith(Responses.Extensions.NoContent());
```

## Asserting requests

Using TestableHttpClient it is also possible to assert that certain requests are acutally made. The following example shows several ways how to assert that requests with authentication headers are made.

```csharp
TestableHttpMessageHandler handler = new();
handler.RespondWith(Sequenced(StatusCode(HttpStatusCode.Unauthenticated), succesResponse));
HttpClient httpClient = handler.CreateClient();
GithubApiClient client = new(httpClient);

_ = await client.GetOrganizationAsync("testablehttpclient");

handler.ShouldHaveMadeRequests().WithRequestHeader("Authorization");
handler.ShouldHaveMadeRequestsTo("*/orgs/testablehttpclient").WithHeader("Authorization", "Bearer *");
```

The starting point for asserting request are the methods `ShouldHaveMadeRequests` and `ShouldHaveMadeRequestsTo`.

`ShouldHaveMadeRequests` doesn't take any arguments and will just count the recorded requests, while `ShouldHaveMadeRequestsTo` requires an URI pattern and will validate that requests made to a specific URI.  
After these initial assertions, extra assertions can be made. Here is a relatively self explainatory list of valid options:
- WithRequestUri
- WithHttpMethod
- WithHttpVersion
- WithHeader
- WithRequestHeader
- WithContentHeader
- WithContent
- WithFormUrlEncodedContent
- WithJsonContent

All methods have an overload where you can specify how many requests are expected. This can be usefull if you expect no requests or a specific number because of some retry mechanism.

The `ShouldHaveMadeRequestsTo` method is basically a shorthand for `handlers.ShouldHaveMadeRequest().WithRequetUri(...)`

## URI Patterns

Both route based responses and the `WithRequestUri` assertion support URI Patterns. URI patterns are based on URI's as specified in [RFC 3986](https://www.rfc-editor.org/rfc/rfc3986), but allow the wildcard character `*` to specify optional parts of an URI.

An URI contains several components:
- The scheme of an URI is optional, but when given it should end with `://`. When not given `*://` is assumed.
- User Information (`username:password@`) is ignored and is not checked at all.
- The host is optional and when not given `*` is assumed. Both IP addresses and registered names are supported.
- The port is optional, but when ':' is provided after host, it should have a value.
- The path is optional, but should start with a `/`. When `/` is given, it can be followed by a `*` to match it with any path.
- Query parameters are optional, when given it should start with a `?`.
- Fragments are ignored, but should start with a `#`.

URI patterns differ from URI's in the following ways:
- Any character is allowed, for example: `myhost:myport` is a valid URI pattern, but not a valid URI. (and this will never match).
- No encoding is performed, and patterns are matched against the unescaped values of an URI.
- Patterns are not normalized, so a path pattern `/test/../example` will not work.

Some examples:

Uri pattern | Matches
------------|--------
\*|Matches any URL
\*://\*/\*?\* | Matches any URL
/get | Matches any URL that uses the path `/get`
http\*://\* | Matches any URL that uses the scheme `http` or `https` (or any other scheme that starts with `http`)
localhost:5000 | Matches any URL that uses localhost for the host and port 5000, no matter what scheme or path is used.

## Modifying behaviour
Some parts of TestableHttpClient can be configured to work differently. This is done via the `Options` property on the `TestableHttpMessageHandler` class.

Currently the following options exist:
- `JsonSerializerOptions`: Options that are used for serializing an deserializing json content.
- `UriPatternMatchingOptions`: Options concerning URI pattern matching, mostly case sensitivity.