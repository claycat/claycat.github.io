---
title: "Deserializing JSON Strings in Test Code"
date: 2024-04-10 12:00:00 +/-TTTT
categories: [Tiketeer]
comments: true
tags: [java, spring, tiketeer]
---

# Introduction

When testing a `RestController` with `MockMvc`, you often need to compare the JSON result with an expected value.

Typically, you use `jsonPath` to verify all the properties of the returned object.

```java
// when - then

	mockMvc.perform(get("/api/members/" + member.getId() + "/sale")
			.contextPath("/api")
			.with(csrf())
			.contentType(MediaType.APPLICATION_JSON)
			.characterEncoding("utf-8")
			.cookie(cookie)
		)
		.andExpect(status().isOk())
		.andExpect(jsonPath("$.data").isArray())
		.andExpect(jsonPath("$.data[0].price").value(1000))
		.andExpect(jsonPath("$.data[0].description").value(""))
		.andExpect(jsonPath("$.data[0].title").value("test"))
		.andExpect(jsonPath("$.data[0].location").value("Seoul"))
		.andExpect(jsonPath("$.data[0].eventTime").value(now.toString()))
		.andExpect(jsonPath("$.data[0].saleStart").value(now.toString()))
		.andExpect(jsonPath("$.data[0].saleEnd").value(now.toString()))
		.andExpect(jsonPath("$.data[0].stock").value(3))
		.andExpect(jsonPath("$.data[0].remainStock").value(1))
		.andExpect(jsonPath("$.data[0].category").value(""))
		.andExpect(jsonPath("$.data[0].runningMinutes").value(600));
```

When the JSON structure is simple, manually verifying with `jsonPath` is convenient.

But as the object grows and includes nested arrays, the fact that `jsonPath` takes plain strings means you can’t rely on IDE assistance, which leads to missed fields or typos, making it cumbersome.

This made me wonder—TypeScript makes it easy to convert JSON strings to objects with `JSON.parse`, so couldn’t we do something similar?

```typescript
const jsonString = '{"name": "John", "age": 30, "city": "New York"}';

interface Person {
  name: string;
  age: number;
  city: string;
}

const person: Person = JSON.parse(jsonString);
```

# Main Content

In our Tiketeer project, we design API response objects using a separate wrapper class:

```java

@Getter
@ToString
@NoArgsConstructor(force = true)
public class ApiResponse<T> {
	private final T data;

	private ApiResponse(T data) {
		this.data = data;
	}

	public static <T> ApiResponse<T> wrap(T data) {
		return new ApiResponse<>(data);
	}
}
```

We use this wrapper class to produce:

1. Single object DTOs
2. Array DTOs (e.g., `TicketingResponse[]`)

I’ll explain how we deserialize each.

### Single Object DTO

Many developers have faced similar problems, so the popular serialization library Jackson already provides a `readValue` method in `ObjectMapper`.

```java
public <T> T readValue(String content, JavaType valueType)
```

Unfortunately, you have to provide a `JavaType`, not just a `Class`.

So first, we’ll create a helper that takes a class and returns the appropriate `JavaType`:

```java
private JavaType getApiResponseType(Class<?> clazz) {
		return objectMapper.getTypeFactory().constructParametricType(ApiResponse.class, clazz);
	}
```

Combining these, you can create a deserialization method like this:

```java

	public <T> ApiResponse<T> getDeserializedApiResponse(String json, Class<T> responseType) throws
		JsonProcessingException {
		return objectMapper.readValue(json, getApiResponseType(responseType));
	}
```

### List Type DTO

If your response returns a list, you can write a method like this.

When returning a `List<TargetType>`, your goal is to get a `JavaType` describing `List<TargetType>`.

Fortunately, `ObjectMapper`’s `TypeFactory` has overloaded `constructParametricType` methods:

![](../../assets/images/2024-06-12-15-54-11.png){: width="500" }

Using this, you can obtain the `JavaType` of `List<TargetType>` as long as you have the `JavaType` for the list element:

```java
private JavaType getApiResponseType(Class<?> clazz) {
	return objectMapper.getTypeFactory().constructParametricType(ApiResponse.class, clazz);
}

// Overload the method used in single object deserialization
private JavaType getApiResponseType(JavaType javaType) {
	return objectMapper.getTypeFactory().constructParametricType(ApiResponse.class, javaType);
}

private JavaType getListApiResponseType(Class<?> clazz) {
	JavaType listType = getListType(clazz);
	return getApiResponseType(listType);
}
```

Now you just need to get the `JavaType` for the List itself:

```java

	private JavaType getListType(Class<?> clazz) {
	return objectMapper.getTypeFactory().constructParametricType(List.class, clazz);
}
```

Putting it together, you get this:

```java
public <T> ApiResponse<List<T>> getDeserializedListApiResponse(String json, Class<T> responseType)
	throws
	JsonProcessingException {
	return objectMapper.readValue(json, getListApiResponseType(responseType));
}
```

In our project, we placed these methods inside a `TestHelper` class, which also handles common repetitive tasks in tests (like database table resets).

Example usage:

```java
	// when - then
	MvcResult result = mockMvc.perform(get("/api/members/" + member.getId() + "/sale")
			.contextPath("/api")
			.with(csrf())
			.contentType(MediaType.APPLICATION_JSON)
			.characterEncoding("utf-8")
			.cookie(cookie)
		)
		.andExpect(status().isOk())
		.andReturn();

	String jsonResult = result.getResponse().getContentAsString();

	ApiResponse<List<GetMemberTicketingSalesResponseDto>> apiResponse =
		testHelper.getDeserializedListApiResponse(
		jsonResult, GetMemberTicketingSalesResponseDto.class);

	var dto = apiResponse.getData().getFirst();

	assertThat(dto.getPrice()).isEqualTo(1000);
	assertThat(dto.getDescription()).isEqualTo("");
	assertThat(dto.getTitle()).isEqualTo("test");
	assertThat(dto.getLocation()).isEqualTo("Seoul");
	assertThat(dto.getEventTime()).isEqualToIgnoringNanos(now);
	assertThat(dto.getSaleStart()).isEqualToIgnoringNanos(now);
	assertThat(dto.getSaleEnd()).isEqualToIgnoringNanos(now);
	assertThat(dto.getStock()).isEqualTo(3);
	assertThat(dto.getRemainStock()).isEqualTo(1);
	assertThat(dto.getCategory()).isEqualTo("");
	assertThat(dto.getRunningMinutes()).isEqualTo(600);
```

### Advantages

Deserializing JSON into objects this way offers another benefit:

You can use DateTime methods directly.

During serialization/deserialization, time values may lose precision or get padded with zeros at the nanosecond level.

With the `jsonPath` approach, it’s difficult to account for these subtle differences when comparing strings.

But when you have objects, you can use methods like `isEqualToIgnoringNanos` to write tests that are both more convenient and precise.

# Conclusion

In this post, we looked at how to deserialize JSON into objects in test code.

For DTOs with many fields or time values including seconds, this approach makes it much easier to write robust tests.

[Code](https://github.com/Tiketeer/Tiketeer-BE/blob/develop/src/test/java/com/tiketeer/Tiketeer/testhelper/TestHelper.java) and [repository](https://github.com/Tiketeer/Tiketeer-BE)

Thank you.
