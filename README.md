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

One of the main features of TestableHttpClient is the ability to stub esponses in order to avoid actual calls to the webserver.

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

### Route base responses
`Reponses.Route` provides a simplified version of Endpoint routing in ASP.NET Core. It can select a response based on the request uri.

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