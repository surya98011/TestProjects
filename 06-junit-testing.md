# 🧪 Module 6: JUnit Testing & Mocking

> All examples use the **ShopEase** e-commerce microservice project.

---

## 📑 Table of Contents

- [1. What is Unit Testing?](#1-what-is-unit-testing)
- [2. JUnit 5 Basics](#2-junit-5-basics)
- [3. Mockito — Isolated Testing](#3-mockito--isolated-testing)
- [4. Testing ShopEase ProductService](#4-testing-shopease-productservice)
- [5. Testing ShopEase ProductController](#5-testing-shopease-productcontroller)
- [6. Code Coverage with JaCoCo](#6-code-coverage-with-jacoco)

---

## 1. What is Unit Testing?

| Concept | Explanation |
|---|---|
| **Unit Testing** | Testing individual components (methods/classes) in isolation |
| **Why?** | Catch bugs early, before code reaches QA team |
| **Industry Standard** | Minimum **80% code coverage** |
| **Who writes?** | Developers (not testers) |

### Testing Pyramid

```
         ┌──────────┐
         │   E2E    │  ← Few (slow, expensive)
        ┌┴──────────┴┐
        │Integration  │  ← Some
       ┌┴────────────┴┐
       │  Unit Tests   │  ← Many (fast, cheap)
       └──────────────┘
```

---

## 2. JUnit 5 Basics

### Key Annotations

| Annotation | Purpose |
|---|---|
| `@Test` | Marks a test method |
| `@BeforeEach` | Runs before each test |
| `@AfterEach` | Runs after each test |
| `@BeforeAll` | Runs once before all tests |
| `@DisplayName` | Custom test name |
| `@Disabled` | Skip this test |

### Key Assertions

```java
assertEquals(expected, actual);          // Check equality
assertNotNull(object);                   // Check not null
assertTrue(condition);                   // Check true
assertThrows(Exception.class, () -> {}); // Check exception
assertAll(() -> {}, () -> {});           // Multiple assertions
```

---

## 3. Mockito — Isolated Testing

### Why Mocking?

```
Controller → depends on → Service → depends on → Repository
```

When testing `ProductService`, we **don't want to call the real database**. We create a **mock** of `ProductRepository`.

### Key Mockito Annotations

| Annotation | Purpose |
|---|---|
| `@Mock` | Create a fake (mock) object |
| `@InjectMocks` | Inject mocks into the class being tested |
| `@ExtendWith(MockitoExtension.class)` | Enable Mockito in JUnit 5 |

---

## 4. Testing ShopEase ProductService

### ProductServiceTest.java

```java
package com.shopease.product.service;

import com.shopease.product.entity.Product;
import com.shopease.product.repository.ProductRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Arrays;
import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class ProductServiceTest {

    @Mock
    private ProductRepository productRepo;  // Fake repository

    @InjectMocks
    private ProductService productService;  // Real service with mock injected

    private Product sampleProduct;

    @BeforeEach
    void setUp() {
        sampleProduct = new Product();
        sampleProduct.setId(1L);
        sampleProduct.setName("Dell Laptop");
        sampleProduct.setPrice(59999.00);
        sampleProduct.setCategory("Laptops");
    }

    @Test
    @DisplayName("Should return all products")
    void testGetAllProducts() {
        // Arrange — Tell mock what to return
        List<Product> mockProducts = Arrays.asList(sampleProduct);
        when(productRepo.findAll()).thenReturn(mockProducts);

        // Act — Call the real service method
        List<Product> result = productService.getAllProducts();

        // Assert — Verify results
        assertNotNull(result);
        assertEquals(1, result.size());
        assertEquals("Dell Laptop", result.get(0).getName());

        // Verify — Confirm the repo method was called exactly once
        verify(productRepo, times(1)).findAll();
    }

    @Test
    @DisplayName("Should return product by ID")
    void testGetProductById() {
        when(productRepo.findById(1L)).thenReturn(Optional.of(sampleProduct));

        Product result = productService.getProductById(1L);

        assertNotNull(result);
        assertEquals("Dell Laptop", result.getName());
        assertEquals(59999.00, result.getPrice());
    }

    @Test
    @DisplayName("Should throw exception for invalid product ID")
    void testGetProductByIdNotFound() {
        when(productRepo.findById(99L)).thenReturn(Optional.empty());

        assertThrows(ProductNotFoundException.class,
                     () -> productService.getProductById(99L));
    }

    @Test
    @DisplayName("Should save new product")
    void testCreateProduct() {
        when(productRepo.save(any(Product.class))).thenReturn(sampleProduct);

        Product result = productService.createProduct(sampleProduct);

        assertNotNull(result);
        assertEquals(1L, result.getId());
        verify(productRepo, times(1)).save(sampleProduct);
    }
}
```

---

## 5. Testing ShopEase ProductController

### ProductControllerTest.java (MockMvc)

```java
package com.shopease.product.controller;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.shopease.product.entity.Product;
import com.shopease.product.service.ProductService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.bean.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.util.Arrays;

import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void testGetAllProducts() throws Exception {
        Product product = new Product(1L, "Dell Laptop", 59999.00, "Laptops");
        when(productService.getAllProducts()).thenReturn(Arrays.asList(product));

        mockMvc.perform(get("/api/products"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$[0].name").value("Dell Laptop"))
               .andExpect(jsonPath("$[0].price").value(59999.00));
    }

    @Test
    void testCreateProduct() throws Exception {
        Product product = new Product(null, "HP Laptop", 49999.00, "Laptops");
        Product saved = new Product(2L, "HP Laptop", 49999.00, "Laptops");
        when(productService.createProduct(any())).thenReturn(saved);

        mockMvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(product)))
               .andExpect(status().isCreated())
               .andExpect(jsonPath("$.id").value(2));
    }
}
```

---

## 6. Code Coverage with JaCoCo

### Add JaCoCo Plugin to `pom.xml`

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.11</version>
            <executions>
                <execution>
                    <goals><goal>prepare-agent</goal></goals>
                </execution>
                <execution>
                    <id>report</id>
                    <phase>test</phase>
                    <goals><goal>report</goal></goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Generate Report

```bash
cd ShopEase/product-service
mvn clean test

# Report generated at: target/site/jacoco/index.html
```

> **💡 Pro Tip:** For new hires in a company, the first 3 months of tasks typically include: (1) Sonar fixes, (2) Writing unit tests, (3) Improving code coverage to 80%+, and (4) Bug fixing.

---

*← [05 — Logging](./05-logging-and-monitoring.md) | [07 — SonarQube →](./07-sonarqube-code-quality.md)*
