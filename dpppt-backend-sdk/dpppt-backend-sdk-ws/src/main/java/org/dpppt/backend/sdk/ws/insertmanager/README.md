# Insert-Manager

## Idea
The Insert-Manager was introduced to reduce logic in controllers. The idea is to provide a second abstraction layer next to the `DataServices` to provide for possible generic validation and normalization. For this there are two mechanisems: modifiers and filters

The Insert-Manager holds a list of `KeyInsertionFilter`, which provide some code, to either filter for invalid data data. Each filter can decide to either skip respective keys, or throw a `InsertException`. Throwing an exception aborts the current insert request, and throws to the controller. Inside the controller the exception can be mapped to a respective error message and http status code.

Additionally the Insert-Manager can be configured to hold a list of `KeyInsertModifier`. Modifiers are can be used to modify incoming keys before inserting into the database. (for example to fix buggy clients)

Encapsulating the logic into smaller pieces of code, should allow for easier and better reviews of the respective filters and modifiers. Further, for each filter or modifier an extensive documentation can be provided, without cluttering the code with too many comments. 

## Valid Keys
A valid key is defined as follows:
- Base64 Encoded key
- Non Fake
- Rolling Period in [1..144]
- Rolling start number inside the configured retention period
- Rolling start number not too far in the future, more precisely not after the day after tomorrow at time of insertion
- Key date must honor the onset date which is given by the health authority


## KeyInsertionFilter Interface
The `KeyInsertionFilter` interface has the following signature:

```java
public interface KeyInsertionFilter {
  List<GaenKey> filter(
      UTCInstant now,
      List<GaenKey> content,
      OSType osType,
      Version osVersion,
      Version appVersion,
      Object principal)
      throws InsertException;
}
```

It gets a `now` object representing _the time the request started_ from the controller , a list of keys, some OS and app related information taken from the `UserAgent` (c.f. `InsertManager@exctractOS` and following) and a possible principal object, representing a authenticated state (e.g. a `JWT`). The function is marked to throw a `InsertException` to stop the inserting process.

## KeyInsertionModifier Interface
The `KeyInsertionModifier` interface has the following signature:

```java
public interface KeyInsertionModifier {
  List<GaenKey> modify(
      UTCInstant now,
      List<GaenKey> content,
      OSType osType,
      Version osVersion,
      Version appVersion,
      Object principal)
      throws InsertException;
}
```

It gets a `now` object representing _the time the request started_ from the controller , a list of keys, some OS and app related information taken from the `UserAgent` (c.f. `InsertManager@exctractOS` and following) and a possible principal object, representing a authenticated state (e.g. a `JWT`). The function is marked to throw a `InsertException` to stop the inserting process.

## Names

The filters should be one of
- `Assert` - lets either pass all keys or throws `InsertException`
- `Remove` - explains which keys are removed
- `Enforce` - explains which keys are kept

## InsertException

An `InsertException` can be thrown inside an implementation of the `InsertionFilter` interface to mark an insert as "unrecoverable" and abort it. Such "unrecoverable" states might be patterns in the uploaded model, which allows for packet sniffing and/or information gathering.

## Default Filters

Looking at the `WSBaseConfig`, we can see that during construction of the `InsertManager` bean, a set of default filters are added:

```java
public abstract class WSBaseConfig implements SchedulingConfigurer, WebMvcConfigurer {

  // ...
    
  @Bean
  public InsertManager insertManager() {
    var manager = new InsertManager(gaenDataService(), gaenValidationUtils());
    manager.addFilter(new AssertBase64(gaenValidationUtils()));
    manager.addFilter(new EnforceMatchingJWTClaims(gaenRequestValidator, gaenValidationUtils()));
    manager.addFilter(new RemoveKeysFromFuture());
    manager.addFilter(new EnforceRetentionPeriod(gaenValidationUtils()));
    manager.addFilter(new RemoveFakeKeys());
    manager.addFilter(new EnforceValidRollingPeriod());
    return manager;
  }
}
```

- `AssertBase64`
    > This filter validates that the key actually is a correctly encoded base64 string. Since we are using 16 bytes of key data, those can be represented with exactly 24 characters. The validation of the length is already done during model validation and is assumed to be correct when reaching the filter. This filter _throws_ a `KeyIsNotBase64Exception` if any of the keys is wrongly encoded. Every key submitted _MUST_ have correct base64 encoding
- `EnforceMatchingJWTClaims`: 
    > This filter compares the supplied keys with information found in the JWT token. During the `exposed` request, the onset date, which will be set by the health authority and inserted as a claim into the JWT is the lower bound for allowed key dates. For the `exposednextday` the JWT contains the previously submitted and checked `delayedKeyDate`, which is compared to the actual supplied key.
- `RemoveKeysFromFuture`: 
    > Representing the maximum allowed time skew. Any key which is further in the future as the day after tomorrow is considered to be _maliciously_ or faulty (for example because of wrong date time settings) uploaded and is hence filtered out.
- `EnforceRetentionPeriod`: 
    > Only keys with key date in the configured retention period are inserted into the datbase. Any key which was valid earlier than `RetentionPeriod` is considered to be outdated and not saved in the database. The key would be removed during the next database clean anyways.
- `RemoveFakeKeys`
    > Only keys that are non-fake are inserted into the database, more precicely keys that have the fake flag set to `0`.
- `EnforceValidRollingPeriod`: 
    > The `RollingPeriod` represents the 10 minutes interval of the key's validity. Negative numbers are not possible, hence any key having a negative rolling period is considered to be _maliciously_ uploaded. Further, according to [Apple/Googles documentation](https://github.com/google/exposure-notifications-server/blob/main/docs/server_functional_requirements.md) values must be in [1..144]


## Additonal Modifiers
- `IOSLegacyProblemRPLT144FModifier`
    > This modifier makes sure, that rolling period is always set to 144. Default value according to EN is 144, so just set it to that. This allows to check for the Google-TEKs also on iOS. Because the Rolling Proximity Identifier is based on the TEK and the unix epoch, this should work. The only downside is that iOS will not be able to optimize verification of the TEKs, because it will have to consider each TEK for a whole day.
- `OldAndroid0RPModifier`: 
    > Some early builds of Google's Exposure Notification API returned TEKs with rolling period set to '0'. According to the specification, this is invalid and will cause both Android and iOS to drop/ignore the key. To mitigate ignoring TEKs from these builds alltogether, the rolling period is increased to '144' (one full day). This should not happen anymore and can be removed in the near future. Until then we are going to log whenever this happens to be able to monitor this problem.



## Configuration 
During construction, instances of `GAENDataService` and `ValidationUtils` are needed. Further, any filter or modifier can be added to the list with `addFilter(KeyInsertionFilter filter)` or `addModifier(KeyInsertionModifier)`. Ideally, this happens inside the [`WSBaseConfig`](../config/WSBaseConfig.java), where default filters are added right after constructing the `InsertManager`. 

To allow for conditional `KeyInsertionFilters` or `KeyInsertionModifiers` refer to the following snippet:

```java
public abstract class WSBaseConfig implements SchedulingConfigurer, WebMvcConfigurer {

  // ...

  @ConditionalOnProperty(
      value = "ws.app.gaen.insertmanager.iosrplt144modifier",
      havingValue = "true",
      matchIfMissing = false)
  @Bean
  public IOSLegacyProblemRPLT144Modifier iosLegacyProblemRPLT144(InsertManager manager) {
    var iosModifier = new IOSLegacyProblemRPLT144Modifier();
    manager.addModifier(iosModifier);
    return iosModifier;
  }
}
```

This looks for a property, either supplied via a `application.properties` file, or via `java` arguments (e.g. `java -D ws.app.gaen.insertmanager.iosrplt144modifier`) and constructs and inserts the respective modifier bean into the modifier chain. For further `SpringBoot` `Conditional` annotations have a look at ["Spring Boot Conditionals"](https://reflectoring.io/spring-boot-conditionals/)

Encapsulating the logic into smaller pieces of code allows for easier and better reviews of the respective filters. 
Further, for each filter an extensive documentation can be provided, without cluttering the code with too many comments. 