# ERD
![](https://github.com/Chocochip101/DB-Study/assets/73146678/301dc47e-bcbb-47bb-bcba-3c76368625e2)

# SQL
1. 카테고리 별로 최저가격인 브랜드와 가격을 조회하고 총액이 얼마인지 확인할 수 있어야 합니다.

```java
@Query("SELECT p.category, b.brandName, p.productPrice.price " +
            "FROM Product p " +
            "JOIN FETCH Brand b on p.brand.id = b.id " +
            "WHERE p.productPrice.price = " +
            "(SELECT MIN(p2.productPrice.price) FROM Product p2 WHERE p2.category = p.category GROUP BY p2.category) ")
List<CategoryMinPrice> findCategoryMinPrice();

```
```java

public CategoryMinPriceResponse getCategoriesMinPrice() {
        List<CategoryMinPrice> categoryMinPrices = productRepository.findCategoryMinPrice().stream()
                .distinct()
                .sorted(Comparator.comparingInt(cp -> cp.getCategory().getOrder()))
                .toList();

        List<CategoryBrandPrice> categoryBrandPrices = categoryMinPrices.stream()
                .map(CategoryBrandPrice::new)
                .toList();

        Long totalPrice = categoryMinPrices.stream()
                .mapToLong(CategoryMinPrice::getPrice)
                .sum();

        return new CategoryMinPriceResponse(categoryBrandPrices, totalPrice);
}
```


2. 단일 브랜드로 전체 카테고리 상품을 구매할 경우 최저가격인 브랜드와 총액이 얼마인지 확인할 수 있어야 합니다.


```java
    @Query("SELECT b.id, b.brandName.value, SUM(p.productPrice.price) " +
            "FROM Brand b " +
            "JOIN Product p ON b.id = p.brand.id " +
            "GROUP BY b.id " +
            "ORDER BY SUM(p.productPrice.price) ASC ")
    List<MinBrandPrice> findBrandByPrices(Pageable pageable);

    @Query("SELECT p.category, p.productPrice.price" +
            "FROM Product p " +
            "WHERE p.brand.id = :brandId")
    List<CategoryInfo> findCategoryInfos(@Param("brandId") Long brandId);
```

```java
    public BrandMinPriceResponse getMinPriceCategoryAndTotal() {
        List<MinBrandPrice> minBrandPrices = brandRepository.findBrandByPrices(firstPageLimit);
        checkMinBrandPricesNotEmpty(minBrandPrices);
        List<CategoryInfo> categoryInfos = brandRepository.findCategoryInfos(minBrandPrices.get(0).getBrandId());

        return new BrandMinPriceResponse(LowestPriceInfo.from(minBrandPrices.get(0), categoryInfos));
    }
```

3. 특정 카테고리에서 최저가격 브랜드와 최고가격 브랜드를 확인하고 각 브랜드 상품의 가격을 확인할 수 있어야 합니다.

```java
 @Query("SELECT b.brandName.value, p.productPrice.price " +
            "FROM Product p " +
            "JOIN FETCH Brand b ON p.brand.id = b.id " +
            "WHERE p.category = :category " +
            "AND p.productPrice.price = (SELECT MIN(p2.productPrice.price) FROM Product p2 WHERE p2.category = :category)")
List<BrandPrice> findMinPriceProductsByCategory(@Param("category") Category category);

@Query("SELECT b.brandName.value, p.productPrice.price " +
            "FROM Product p " +
            "JOIN FETCH Brand b ON p.brand.id = b.id " +
            "WHERE p.category = :category " +
            "AND p.productPrice.price = (SELECT MAX(p2.productPrice.price) FROM Product p2 WHERE p2.category = :category)")
List<BrandPrice> findMaxPriceProductsByCategory(@Param("category") Category category);
```

