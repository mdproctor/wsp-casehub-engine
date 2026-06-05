# The CI That Cried Wolf

Twenty-four hours chasing a test hang that didn't exist.

Every CI run showed the same pattern: output stopped at `JpaEventLogRepositoryTest`, silence for 45 minutes, timeout. Five consecutive runs, same freeze point. The obvious diagnosis — that test class is deadlocking on the Vert.x event loop when 10 concurrent threads fight over 4 event loop threads on a 2-core CI runner.

Except `JpaEventLogRepositoryTest` passed every single time. 26 tests, zero failures, 1.068 seconds. We know this because the Surefire reports artifact — uploaded even on cancelled runs thanks to `if: always()` — contained the `.txt` files proving it. The CI log was lying.

The real problem was two modules later in the build order: `actor-state`. It had `@QuarkusTest` classes but no `quarkus-maven-plugin`. Without the plugin's `generate-code` goals, Quarkus augmentation runs inside Surefire's forked test JVM at test time. Locally, on an 8-core Mac, augmentation takes 2 seconds. On a 2-core GitHub Actions runner processing the full engine+work+qhorus+ledger CDI graph — hundreds of beans, 15 extensions — it hangs indefinitely with no error output.

The misdirection was elegant. Hibernate SQL tracing (`log.sql=true` + `log.bind-parameters=true`) in `persistence-hibernate` generated thousands of TRACE-level log lines. These filled the 64KB stdout pipe buffer between Surefire's forked JVM and the parent Maven process. Once the buffer filled, subsequent Maven output — including "Building casehub-engine-actor-state" — never reached the CI log. The build continued silently, and the last visible line happened to be the verbose module's test class.

The fix was one XML block:

```xml
<plugin>
    <groupId>io.quarkus.platform</groupId>
    <artifactId>quarkus-maven-plugin</artifactId>
    <extensions>true</extensions>
    <executions>
        <execution>
            <goals>
                <goal>generate-code</goal>
                <goal>generate-code-tests</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Build time: 11 minutes 38 seconds. Faster than before the PR.

Three lessons from this:

**Check the artifact, not the log.** CI logs are a stream — they can be truncated, buffered, or silenced by `redirectTestOutputToFile`. The Surefire reports artifact is ground truth. `gh run download <id> --name surefire-reports` and check which modules have report directories. Present = completed. Absent = hung.

**A module that passes locally on 8 cores can hang forever on 2.** `quarkus-maven-plugin` moves augmentation from test time to compile time. Without it, the augmentation phase runs inside the forked JVM where `forkedProcessTimeoutInSeconds` doesn't cover it and `@Timeout` annotations can't reach it. The failure mode is silent — no error, no timeout, no stack trace.

**Actor-state was never tested in CI.** The Jun 3 "successful" build had 14 modules in the reactor — actor-state wasn't one of them. The module sat in the pom.xml untested until our PR happened to be the first that actually built it. The Surefire reports from that successful build confirmed it: no `actor-state` directory.
