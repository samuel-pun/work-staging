To generate a CSV file with dynamic columns based on a `Map<String, Integer>` using OpenCSV, you can use the `@CsvBindAndJoinByName` annotation in combination with a customized `HeaderColumnNameMappingStrategy`.

The standard `HeaderColumnNameMappingStrategy` typically generates headers only for statically defined fields. By subclassing it and overriding `generateHeader`, you can inject the dynamic column names (keys from your map) at runtime.

### Core Solution

The solution involves three steps:
1.  **Annotate the Bean**: Use `@CsvBindAndJoinByName` for the Map field.
2.  **Custom Strategy**: Create a simple subclass of `HeaderColumnNameMappingStrategy` to force the inclusion of dynamic headers.
3.  **Write**: Initialize the strategy with the full list of headers (static + dynamic) and write.

### Code Snippet

```java
import com.opencsv.bean.*;
import com.opencsv.exceptions.CsvDataTypeMismatchException;
import com.opencsv.exceptions.CsvRequiredFieldEmptyException;
import org.apache.commons.collections4.MultiValuedMap;
import org.apache.commons.collections4.multimap.ArrayListValuedHashMap;

import java.io.FileWriter;
import java.io.Writer;
import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

// 1. Define the Bean
public class DynamicProduct {

    @CsvBindByName(column = "productId")
    private String productId;

    @CsvBindByName(column = "name")
    private String name;

    // Matches any column name that isn't caught by other annotations
    // elementType defines the type of the Map values (Integer in your case)
    @CsvBindAndJoinByName(column = ".*", elementType = Integer.class)
    private MultiValuedMap<String, Integer> attributes = new ArrayListValuedHashMap<>();

    public DynamicProduct(String productId, String name) {
        this.productId = productId;
        this.name = name;
    }

    public void addAttribute(String key, Integer value) {
        this.attributes.put(key, value);
    }
    
    // Getters are required for OpenCSV writing
    public String getProductId() { return productId; }
    public String getName() { return name; }
    public MultiValuedMap<String, Integer> getAttributes() { return attributes; }
}

// 2. Define the Custom Strategy
class DynamicHeaderStrategy<T> extends HeaderColumnNameMappingStrategy<T> {
    private final String[] header;

    public DynamicHeaderStrategy(Class<T> type, String[] header) {
        this.header = header;
        setType(type); // Initialize the strategy with the bean type
    }

    @Override
    public String[] generateHeader(T bean) throws CsvRequiredFieldEmptyException {
        // Return the explicit header list we calculated at runtime
        return header;
    }
}

// 3. Execution Logic
public class CsvWriterDemo {
    public static void main(String[] args) throws Exception {
        // Create sample data
        DynamicProduct p1 = new DynamicProduct("P001", "Widget");
        p1.addAttribute("weight", 100);
        p1.addAttribute("height", 50);

        DynamicProduct p2 = new DynamicProduct("P002", "Gadget");
        p2.addAttribute("weight", 120);
        p2.addAttribute("depth", 30); // Different attribute

        List<DynamicProduct> products = Arrays.asList(p1, p2);

        // 4. Determine all dynamic headers at runtime
        Set<String> dynamicKeys = new LinkedHashSet<>();
        for (DynamicProduct p : products) {
            dynamicKeys.addAll(p.getAttributes().keySet());
        }

        // Combine static headers with dynamic keys
        // Note: Order matters. Standard headers first, then dynamic.
        String[] allHeaders = Stream.concat(
            Stream.of("productId", "name"), // Static columns
            dynamicKeys.stream()            // Dynamic columns
        ).toArray(String[]::new);

        // 5. Write to CSV
        try (Writer writer = new FileWriter("dynamic_output.csv")) {
            
            // Initialize custom strategy with the calculated headers
            DynamicHeaderStrategy<DynamicProduct> strategy = 
                new DynamicHeaderStrategy<>(DynamicProduct.class, allHeaders);

            StatefulBeanToCsv<DynamicProduct> beanToCsv = new StatefulBeanToCsvBuilder<DynamicProduct>(writer)
                    .withMappingStrategy(strategy)
                    .withQuotechar(com.opencsv.CSVWriter.NO_QUOTE_CHARACTER)
                    .build();

            beanToCsv.write(products);
        }
    }
}
```

### Key Implementation Details
*   **`@CsvBindAndJoinByName`**: The `column` parameter takes a regex pattern. `.*` acts as a catch-all for any column name provided by the strategy that doesn't match a specific `@CsvBindByName` field.[1][2]
*   **MultiValuedMap**: OpenCSV 5.x typically requires Apache Commons `MultiValuedMap` for join mappings rather than a standard `java.util.Map` to handle potential duplicate headers, though it works similarly for unique keys.[2][3]
*   **Strategy Override**: The default `HeaderColumnNameMappingStrategy` cannot "see" your map keys as headers automatically. Overriding `generateHeader` bridges this gap by feeding the complete list of runtime columns to the writer.[4][5]

來源
[1] CsvBindAndJoinByName (opencsv 5.12.0 API) https://opencsv.sourceforge.net/apidocs/com/opencsv/bean/CsvBindAndJoinByName.html
[2] Read CSV files with known and unknown columns in java https://stackoverflow.com/questions/57034570/read-csv-files-with-known-and-unknown-columns-in-java
[3] csv导入导出(opencsv) 原创 - CSDN博客 https://blog.csdn.net/qq_41609208/article/details/111461171
[4] How to Create CSV File from POJO with Custom Column Headers ... https://www.baeldung.com/java-create-csv-pojo-customize-columns
[5] Writing csv file with OpenCsv without capitalized headers ... https://dev.to/franzwong/writing-csv-file-with-opencsv-without-capitalized-headers-and-follows-declaration-order-207e
[6] Supported Dynamic Number of Columns for Collection Field? #358 https://github.com/uniVocity/univocity-parsers/issues/358
[7] opencsv – https://opencsv.sourceforge.net
[8] OpenCSV - How to map selected columns to Java Bean regardless ... https://stackoverflow.com/questions/13505653/opencsv-how-to-map-selected-columns-to-java-bean-regardless-of-order
[9] Mapping Java Beans to CSV Using OpenCSV Library https://www.randomcodez.com/java-mapping-java-beans-to-csv-using-opencsv
[10] java如何设置scv列名,OpenCSV - 如何将选定的列映射到Java Bean https://blog.csdn.net/weixin_33308507/article/details/118838264
[11] Mapping Java Beans to CSV Using OpenCSV - GeeksforGeeks https://www.geeksforgeeks.org/java/mapping-java-beans-to-csv-using-opencsv/
[12] MappingStrategy (opencsv 5.12.0 API) https://opencsv.sourceforge.net/apidocs/com/opencsv/bean/MappingStrategy.html
[13] CsvBindAndJoinByName (opencsv 4.6 API) - javadoc.io https://javadoc.io/doc/com.opencsv/opencsv/4.6/com/opencsv/bean/CsvBindAndJoinByName.html
[14] ColumnPositionMappingStrategy.java - opencsv – https://opencsv.sourceforge.net/jacoco/com.opencsv.bean/ColumnPositionMappingStrategy.java.html
[15] ColumnPositionMappingStrategy (opencsv 5.11 API) https://javadoc.io/doc/com.opencsv/opencsv/5.11/com/opencsv/bean/ColumnPositionMappingStrategy.html
[16] CsvBindAndJoinByName xref - opencsv – https://opencsv.sourceforge.net/xref/com/opencsv/bean/CsvBindAndJoinByName.html
[17] opencsv HeaderColumnNameAndOrderMappingStrategy · GitHub https://gist.github.com/ammmze/ec0334d107cb63c586ffd8fc51ec5757
[18] Mapping CSV to JavaBeans Using OpenCSV https://www.geeksforgeeks.org/java/mapping-csv-to-javabeans-using-opencsv/
[19] Solved: How to Map Dynamic Columns from an Input CSV file https://community.qlik.com/t5/Talend-Studio/How-to-Map-Dynamic-Columns-from-an-Input-CSV-file/td-p/2323956
[20] Introduction to OpenCSV | Baeldung https://www.baeldung.com/opencsv
[21] ColumnPositionMappingStrategy xref https://opencsv.sourceforge.net/xref/com/opencsv/bean/ColumnPositionMappingStrategy.html
[22] Write Bean values to a CSV file using OpenCSV - Coderanch https://coderanch.com/t/614171/open-source/Write-Bean-values-CSV-file
[23] OpenCSV: How to create CSV file from POJO with custom column ... https://stackoverflow.com/questions/45203867/opencsv-how-to-create-csv-file-from-pojo-with-custom-column-headers-and-custom
[24] How to add dynamic column and value in csv using opencsv https://stackoverflow.com/questions/60003935/how-to-add-dynamic-column-and-value-in-csv-using-opencsv
[25] Map to CSV Using OpenCSV - hub4techie - WordPress.com https://anuragdeb3.wordpress.com/2024/01/11/map-to-csv-using-opencsv/
[26] Read / Write CSV files in Java using OpenCSV - CalliCoder https://www.callicoder.com/java-read-write-csv-file-opencsv/
[27] ColumnPositionMappingStrategy (opencsv 5.12.0 API) https://opencsv.sourceforge.net/apidocs/index.html?com%2Fopencsv%2Fbean%2FColumnPositionMappingStrategy.html
[28] MappingStragety to write a field of type hashMap https://stackoverflow.com/questions/45398784/mappingstragety-to-write-a-field-of-type-hashmap
[29] opencsv HeaderColumnNameAndOrderMappingStrategy https://gist.github.com/ammmze/ec0334d107cb63c586ffd8fc51ec5757?permalink_comment_id=3562771
[30] AbstractMappingStrategy (opencsv 5.5.1 API) https://javadoc.io/doc/com.opencsv/opencsv/5.5.1/com/opencsv/bean/AbstractMappingStrategy.html
[31] How to Write Hashmap to CSV File https://www.baeldung.com/java-write-hashmap-csv
[32] How to read and write CSV files using OpenCSV https://attacomsian.com/blog/read-write-csv-files-opencsv
