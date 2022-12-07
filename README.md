# TestableHttpClient

Using HttpClient in code that is unit tested is seen as rather difficult, mainly because HttpClient itself can't be easily mocked and a lot of extension methods are used.  
TestableHttpClient aims to make it easier to assert the calls that are made via an HttpClient and to stub the responses that are being returned.

## How to install

TestableHttpClient is a NuGet package, either search for "TestableHttpClient" in the NuGet Package Manger of Visual Studio, or perform the following command:

```pwsh
dotnet add package TestableHttpClient
```

## Basic usage

When you want to test code that makes calls to a webserver, but don't want to rely on that server, you can use TestableHttpClient to create a stub.
The following example shows the basic concept:

```csharp
TestableHttpMessageHandler handler = new();
handler.RespondWith(Responses.StatusCode(HttpStatusCode.Unauthorized));

HttpClient client = handler.CreateClient();

HttpResponseMessage response = await client.GetAsync("https://github.com/users");

Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
```