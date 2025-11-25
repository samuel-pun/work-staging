You can generate the 200 scenarios by building the header and value strings dynamically from a `scenarioCount` using streams, so you never have to hardcode `"Scenario ID1,...,Scenario ID200"` or the values. This fits well with your existing work on dynamic CSV columns in Java.

## Core idea

Instead of hardcoding `EXPECTED_HEADER` and `EXPECTED_VALUES`, define a `scenarioCount` (e.g. 200) and use `IntStream.rangeClosed(1, scenarioCount)` to create both the `"Scenario IDX"` header part and the comma‑separated rounded values.  
If you already have an `STV_MAP<Integer, BigDecimal>` filled with 1..200, you only need to generate the two strings from that map; if not, you can also generate the map programmatically.

## Code example

```java
import static java.util.stream.Collectors.joining;
import static java.util.stream.Collectors.toMap;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.IntStream;

@Test
void someTest() {
    final int SCENARIO_COUNT = 200;

    // Example: generate STV_MAP for 1..200 (replace value function with your real logic)
    final Map<Integer, BigDecimal> STV_MAP =
        IntStream.rangeClosed(1, SCENARIO_COUNT)
                 .boxed()
                 .collect(toMap(
                     Function.identity(),
                     i -> BigDecimal.valueOf(100).add(BigDecimal.valueOf(i).multiply(BigDecimal.valueOf(0.5))),
                     (a, b) -> a,
                     LinkedHashMap::new
                 ));

    final String EXPECTED_HEADER =
        IntStream.rangeClosed(1, SCENARIO_COUNT)
                 .mapToObj(i -> "Scenario ID" + i)
                 .collect(joining(","));

    final String EXPECTED_VALUES =
        IntStream.rangeClosed(1, SCENARIO_COUNT)
                 .mapToObj(i -> STV_MAP.get(i)
                     .setScale(0, RoundingMode.HALF_UP)   // 111.1 -> 111, 333.9 -> 334
                     .toPlainString())
                 .collect(joining(","));

    CsvAssertionForStvUtil.assertCsvExact(
        outFilePath.toFile(),
        "Date,Run Type,Run Number,Created Date,Created Time,Batch Processing Group," +
            "DCASS Participant,FIRM ID,PART ID,AC TYPE,AC No,Currency," +
            EXPECTED_HEADER,
        List.of(
            "20251125,NORMAL,1,20251125,103620,SEOCH," +
                "DCASS_PART,CCMS_FIRM,CCMS_PART,CCMS_AC_TYPE,CCMS_AC_NO,HKD," +
                EXPECTED_VALUES
        )
    );
}
```

If your production code already produces `STV_MAP` for 1..200, you can drop the `STV_MAP` construction above and keep only the `EXPECTED_HEADER` and `EXPECTED_VALUES` generation using the same `SCENARIO_COUNT`.

