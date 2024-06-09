# NestJS MurLock

MurLock is a distributed lock solution designed for the NestJS framework. It provides a decorator `@MurLock()` that allows for critical sections of your application to be locked to prevent race conditions. MurLock uses Redis to ensure locks are respected across multiple instances of your application, making it perfect for microservices.

## Features

- **Redis-Based**: Implements a fast and effective lock mechanism using Redis.
- **Parameter-Based Locking**: Creates locks based on request parameters or bodies.
- **Highly Customizable**: Customize many parameters, such as lock duration.
- **Retry Mechanism**: Implements an exponential back-off strategy if the lock is not obtained.
- **Logging: Provides**: logging options for debugging and monitoring.
- **OOP and Generic Structure**: Easily integratable and expandable due to its OOP and generic design.

## Installation

MurLock has a peer dependency on `@nestjs/common` and `reflect-metadata`. These should already be installed in your NestJS project. In addition, you'll also need to install the `redis` package.

```bash
npm install --save murlock redis reflect-metadata
```

## Basic Usage

MurLock is primarily used through the `@MurLock()` decorator.

First, you need to import the `MurLockModule` and set it up in your module using `forRoot`. This method is used for global configuration that can be reused across different parts of your application.

```typescript
import { MurLockModule } from 'murlock';

@Module({
  imports: [
    MurLockModule.forRoot({
      redisOptions: { url: 'redis://localhost:6379' },
      wait: 1000,
      maxAttempts: 3,
      logLevel: 'log',
      ignoreUnlockFail: false,
    }),
  ],
})
export class AppModule {}
```

Then, you can use `@MurLock()` in your services:

```typescript
import { MurLock } from 'murlock';

@Injectable()
export class AppService {
  @MurLock(5000, 'user.id')
  async someFunction(user: User): Promise<void> {
    // Some critical section that only one request should be able to execute at a time
  }
}
```

By default, if there is single wrapped parameter, the property of parameter can be called directly as it shown.

```typescript
import { MurLock } from 'murlock';

@Injectable()
export class AppService {
  @MurLock(5000, 'userId')
  async someFunction({ userId, firstName, lastName }: { userId: string, firstName: string, lastName: string} ): Promise<void> {
    // Some critical section that only one request should be able to execute at a time
  }
}
```

If there are multiple wrapped parameter, you can call it by {index of parameter}.{parameter name} as it shown

```typescript
import { MurLock } from 'murlock';

@Injectable()
export class AppService {
  @MurLock(5000, '0.userId', '1.transactionId')
  async someFunction({ userId, firstName, lastName }: UserDTO, { balance, transactionId }: TransactionDTO ): Promise<void> {
    // Some critical section that only one request should be able to execute at a time
  }
}
```

In the example above, the `@MurLock()` decorator will prevent `someFunction()` from being executed concurrently for the same user. If another request comes in for the same user before `someFunction()` has finished executing, it will wait up to 5000 milliseconds (5 seconds) for the lock to be released. If the lock is not released within this time, an `MurLockException` will be thrown.

The parameters to `@MurLock()` are a release time (in milliseconds), followed by any number of key parameters. The key parameters are used to create a unique key for each lock. They should be properties of the parameters of the method. In the example above, 'user.id' is used, which means the lock key will be different for each user ID.

## Advanced Usage

MurLock also supports async configuration. This can be useful if your Redis configuration is not known at compile time.

```typescript
import { MurLockModule } from 'murlock';

@Module({
  imports: [
    MurLockModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        redisOptions: configService.get('REDIS_OPTIONS'),
        wait: configService.get('MURLOCK_WAIT'),
        maxAttempts: configService.get('MURLOCK_MAX_ATTEMPTS'),
        logLevel: configService.get('LOG_LEVEL'),
        ignoreUnlockFail: configService.get('LOG_LEVEL')
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

In the example above, the `ConfigModule` and `ConfigService` are used to provide the configuration for MurLock asynchronously.

For more details on usage and configuration, please refer to the API documentation below.

## Using Custom Lock Key

By default, murlock use class and method name prefix for example Userservice:createUser:{userId}. By setting lockKeyPrefix as 'custom' you can define by yourself manually.

```typescript
import { MurLockModule } from 'murlock';

@Module({
  imports: [
    MurLockModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        redisOptions: configService.get('REDIS_OPTIONS'),
        wait: configService.get('MURLOCK_WAIT'),
        maxAttempts: configService.get('MURLOCK_MAX_ATTEMPTS'),
        logLevel: configService.get('LOG_LEVEL'),
        lockKeyPrefix: 'custom'
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

```typescript
import { MurLock } from 'murlock';

@Injectable()
export class AppService {
  @MurLock(5000, 'someCustomKey', 'userId')
  async someFunction(userId): Promise<void> {
    // Some critical section that only one request should be able to execute at a time
  }
}
```

### Ignoring Unlock Failures

In some scenarios, throwing an exception when a lock cannot be released can be undesirable. For example, you might prefer to log the failure and continue without interrupting the flow of your application. To enable this behavior, set the `ignoreUnlockFail` option to `true` in your configuration:

```typescript
import { MurLockModule } from 'murlock';

MurLockModule.forRoot({
  redisOptions: { url: 'redis://localhost:6379' },
  wait: 1000,
  maxAttempts: 3,
  logLevel: 'log',
  ignoreUnlockFail: true, // Unlock failures will be logged instead of throwing exceptions.
  lockKeyPrefix: 'default' // optional, use 'default' if you would like to lock keys as servicename:methodname:customdata, otherwise use 'custom' to manually write each lock key
}),
```

If we assume userId as 65782628 Lockey here will be someCustomKey:65782628

## Using `MurLockService` Directly

While the `@MurLock()` decorator provides a convenient and declarative way to handle locking within your NestJS application, there may be cases where you need more control over the lock lifecycle. For such cases, `MurLockService` offers a programmatic way to manage locks, allowing for fine-grained control over the lock and unlock process through the `runWithLock` method.

### Injecting `MurLockService`

First, inject `MurLockService` into your service:

```typescript
import { Injectable } from '@nestjs/common';
import { MurLockService } from 'murlock';

@Injectable()
export class YourService {
  constructor(private murLockService: MurLockService) {}
  
  // Your methods where you want to use the lock
}
```

#### Acquiring a Lock

You no longer need to manually manage `lock` and `unlock`. Instead, use the `runWithLock` method, which handles both acquiring and releasing the lock:

```typescript
async performTaskWithLock() {
  const lockKey = 'unique_lock_key';
  const lockTime = 3000; // Duration for which the lock should be held, in milliseconds

  try {
    await this.murLockService.runWithLock(lockKey, lockTime, async () => {
      // Proceed with the operation that requires the lock
    });
  } catch (error) {
    // Handle the error if the lock could not be acquired or any other exceptions
    throw error;
  }
}
```

### Handling Errors

The `runWithLock` method throws an exception if the lock cannot be acquired within the specified time or if an error occurs during the execution of the function:

```typescript
try {
  await this.murLockService.runWithLock(lockKey, lockTime, async () => {
    // Locked operations
  });
} catch (error) {
  // Error handling logic
}
```

Directly using `MurLockService` gives you finer control over lock management but also increases the responsibility to ensure locks are correctly managed throughout your application's lifecycle.

---

This refined section is suitable for developers looking for documentation on using `MurLockService` directly in their projects and adheres to the typical conventions found in README files for open-source projects.

## Best Practices and Considerations

- **Short-lived Locks**: Ensure that locks are short-lived to prevent deadlocks and to increase the efficiency of your application.
**Error Handling**: Robustly handle errors during lock acquisition:
  - **Graceful Failures**: If a lock cannot be obtained, handle the situation gracefully, potentially logging the incident and retrying the operation.
  - **Consider Failures in Unlocking**: Even with `ignoreUnlockFail` set to true, implement error handling strategies to log and manage unlock failures, ensuring they do not disrupt the application flow.
- **Logging**: Adjust the `logLevel` based on your environment. Use 'debug' for development and 'error' or 'warn' for production.
- **Consistency**: Use consistent lock keys that clearly represent the resources or operations they are meant to protect.
- **Customizable Lock Keys**: Utilize the `lockKeyPrefix` to tailor how lock keys are constructed:
  - **Default**: Automatically includes the class and method name, e.g., `Userservice:createUser:{userId}`.
  - **Custom**: Set `lockKeyPrefix` to 'custom' and define lock keys explicitly to fine-tune lock scope and granularity.
- **Resource Cleanup**: Even though `runWithLock` manages lock cleanup, ensure your application logic correctly handles any necessary cleanup or rollback in case of errors.
- **Use of Finally Block**: Explicitly manage lock release in a `finally` block to ensure that locks are always released, preventing potential deadlocks and resource leaks.

## API Documentation

### MurLock(releaseTime: number, ...keyParams: string[])

A method decorator to indicate that a particular method should be locked.

- `releaseTime`: Time in milliseconds after which the lock should be automatically released.
- `...keyParams`: Method parameters based on which the lock should be made. The format is `paramName.attribute`. If just `paramName` is provided, it will use the `toString` method of that parameter.

### Configuration Options

Here are the customizable options for `MurLockModule`, allowing you to tailor its behavior to best fit your application's needs:

- **redisOptions:** Configuration settings for the Redis client, such as the connection URL.
- **wait:** Time in milliseconds to wait before retrying to obtain a lock if the initial attempt fails.
- **maxAttempts:** The maximum number of attempts to try and acquire a lock before giving up.
- **logLevel:** Determines the level of logging used within the module. Options include 'none', 'error', 'warn', 'log', or 'debug'.
- **ignoreUnlockFail (optional):** When set to `true`, the module will not throw an exception if releasing a lock fails. This setting helps in scenarios where failing silently is preferred over interrupting the application flow. Defaults to `false` to ensure that failures are noticed and handled appropriately.
- **lockKeyPrefix (optional)**: Specifies how lock keys are prefixed, allowing for greater flexibility:
  - **Default**: Uses class and method names as prefixes, e.g., `Userservice:createUser:{userId}`.
  - **Custom**: Set this to 'custom' to define lock keys manually in your service methods, allowing for specific lock key constructions beyond the standard naming.

### MurLockService

A NestJS injectable service to interact with the locking mechanism directly.

## Limitations

- **Redis Persistence:** Ensure that your Redis instance has RDB persistence enabled. This ensures that in case of a crash, locks are not lost.
- **Single Redis Instance:** MurLock is not designed to work with Redis cluster mode. It's essential to ensure that locks are always set to a single instance.

## Contributions & Support

We welcome contributions! Please see our contributing guide for more information. For support, raise an issue on our GitHub repository.

## License

This project is licensed under the [MIT License](https://github.com/felanios/murlock/blob/master/LICENSE).

## Contact

If you have any questions or feedback, feel free to contact me at <ozmen.eyupfurkan@gmail.com>.

---

We hope you find MurLock useful in your projects. Don't forget to star our repo if you find it helpful!

Happy coding! 🚀
