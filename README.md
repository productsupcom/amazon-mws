# Amazon Marketplace Webservices

Interaction with the Amazon Api for vendors called MWS

### Change your composer.json file 

```
"require": {
    // ...
    "mcs/amazon-mws": "*",
    // ...
},
"repositories": [
    {
        "name": "mws/amazon-mws",
        "type": "git",
        "url": "git@github.com:productsupcom/amazon-mws.git"
    }
]
```

### Installation:
```bash
$ composer require mcs/amazon-mws
```

### Initiate the client

```php
require_once 'vendor/autoload.php';

$client = new MCS\MWSClient([
    'Marketplace_Id' => '',
    'Seller_Id' => '',
    'Access_Key_ID' => '',
    'Secret_Access_Key' => '',
    'MWSAuthToken' => '' // Optional. Only use this key if you are a third party user/developer
]);

// Optionally check if the supplied credentials are valid
if ($client->validateCredentials()) {
    // Credentials are valid
} else {
    // Credentials are not valid
}
```

### Get orders

```php
$fromDate = new DateTime('2016-01-01');
$orders = $client->ListOrders($fromDate);
foreach ($orders as $order) {
    $items = $client->ListOrderItems($order['AmazonOrderId']);
    print_r($order);
    print_r($items);
}
```

### Get product attributes

```php
$searchField = 'ASIN'; // Can be GCID, SellerSKU, UPC, EAN, ISBN, or JAN

$result = $client->GetMatchingProductForId([
    '<ASIN1>', '<ASIN2>', '<ASIN3>'
], $searchField);

print_r($result);
```

### Create or update a marketplace product

```php
public function uploadAmazon(string $productType, Product $product, int $productNode, $otherAttributes = []) 
{
    $client = new MCS\MWSClient([
        'Marketplace_Id' => '',
        'Seller_Id' => '',
        'Access_Key_ID' => '',
        'Secret_Access_Key' => '',
        'MWSAuthToken' => '' // Optional. Only use this key if you are a third party user/developer
    ]);
    $amazonProducts = [];
    $postItems = [];
    $amazonProduct = new AmazonMarketPlaceProduct();
    if ($product->productSku) {
        $amazonProduct->setSku($product->parent_sku)
            ->setFeedProductType($productType)
            ->setBrand($product->brand)
            ->setTitle($product->title)
            ->setManufacturer($product->manufacturer)
            ->setRecommendedBrowseNodes($productNode)
            ->setParentChild('Parent')
            ->setOtherAttributes(\app\core\helpers\ArrayHelper::clearValue($otherAttributes))
            ->setVariationTheme($product->variation_theme);
        array_push($amazonProducts, $amazonProduct);
        array_push($postItems, $amazonProduct->toArray(false));

        foreach ($product->productSku as $productSku) {
            $retailPrice = $productSku->retail_price ?: 0;
            $salePrice = $productSku->sale_price ?: 0;
            $saleDate = $salePrice ? explode('~', $productSku->sale_date) : [];

            $_amazonProduct = clone $amazonProduct;
            $_amazonProduct->setSku($productSku->product_sku)
                ->setFeedProductType($productType)
                ->setBrand($product->brand)
                ->setTitle($product->title)
                ->setManufacturer($product->manufacturer)
                ->setPrice($retailPrice > 0 ? CurrencyConverter::CNYConverter($currency, $retailPrice) : '')
                ->setSalePrice($salePrice > 0 ? CurrencyConverter::CNYConverter($currency, $salePrice) : '')
                ->setProductId($amazonProductId)
                ->setSizeName($productSku->size ?: '')
                ->setColorName($productSku->color ?: '')
                ->setProductIdType('EAN')
                ->setCurrency($currency)
                ->setConditionType('New')
                ->setWeight($product->weight)
                ->setQuantity($product->quantity)
                ->setParentChild('Child')
                ->setParentSku($parentSku)
                ->setSaleFromDate(count($saleDate) == 2 ? $saleDate[0] : '')
                ->setSaleEndDate(count($saleDate) == 2 ? $saleDate[1] : '')
                ->setVariationTheme($product->variation_theme)
                ->setKeywords($product->search_keyword)
                ->setRecommendedBrowseNodes($productNode)
                ->setBulletPoint($features)
                ->setDescription($product->description)
                ->setOtherAttributes($otherAttributes)
                ->setImage($productSku->images);
            array_push($amazonProducts, $_amazonProduct);
            array_push($postItems, $_amazonProduct->toArray(false));
        }
    } else {
        $retailPrice = $product->price ?: 0;
        $salePrice = $product->sale_price ?: 0;
        $saleDate = $salePrice ? explode('~', $product->sale_date) : [];

        $amazonProduct->setSku($parentSku)
            ->setFeedProductType($productType)
            ->setBrand($product->brand)
            ->setTitle($product->{$titleAttribute})
            ->setManufacturer($product->manufacturer)
            ->setPrice($retailPrice > 0 ? CurrencyConverter::CNYConverter($currency, $retailPrice) : '')
            ->setSalePrice($salePrice > 0 ? CurrencyConverter::CNYConverter($currency, $salePrice) : '')
            ->setProductId($amazonProductId)
            ->setProductIdType('EAN')
            ->setCurrency($currency)
            ->setConditionType('New')
            ->setWeight($product->weight)
            ->setQuantity($product->quantity)
            ->setSaleFromDate(count($saleDate) == 2 ? $saleDate[0] : '')
            ->setSaleEndDate(count($saleDate) == 2 ? $saleDate[1] : '')
            ->setKeywords($product->search_keyword)
            ->setRecommendedBrowseNodes($productNode)
            ->setBulletPoint($features)
            ->setDescription($product->description)
            ->setOtherAttributes($otherAttributes)
            ->setImage($product->images);
        array_push($amazonProducts, $amazonProduct);
        array_push($postItems, $amazonProduct->toArray(false));
    }

    // You can also submit an array of MWSProduct objects
    $feed = $client->postProduct(
        $amazonProducts,
        'fptcustomcustom',
        '2019.0501',
        'SE9NRV9MSUdIVElOR19BTkRfTEFNUFM='
    );
    return $feed;
}
```


### Update product stock

```php
$result = $client->updateStock([
    'sku1' => 20,
    'sku2' => 9,
]);
print_r($result);

$info = $client->GetFeedSubmissionResult($result['FeedSubmissionId']);
print_r($info);
```

### Update product stock with fulfillment latency specified

```php
$result = $client->updateStockWithFulfillmentLatency([
    ['sku' => 'sku1', 'quantity' => 20, 'latency' => 1],
    ['sku' => 'sku2', 'quantity' => 20, 'latency' => 1],
]);
print_r($result);

$info = $client->GetFeedSubmissionResult($result['FeedSubmissionId']);
print_r($info);
```

### Update product pricing

```php
$result = $client->updatePrice([
    'sku1' => '20.99',
    'sku2' => '100.00',
]);
print_r($result);

$info = $client->GetFeedSubmissionResult($result['FeedSubmissionId']);
print_r($info);
```

### Reports

For all report types, visit:  [http://docs.developer.amazonservices.com](http://docs.developer.amazonservices.com/en_US/reports/Reports_ReportType.html#ReportTypeCategories__ListingsReports)

```php
$reportId = $client->RequestReport('_GET_MERCHANT_LISTINGS_DATA_');
// Wait a couple of minutes and get it's content
$report_content = $client->GetReport($reportId);
print_r($report_content);
```
### Available methods

View source for detailed argument description.
All methods starting with an uppercase character are also documented in the [Amazon MWS documentation](http://docs.developer.amazonservices.com/en_US/dev_guide/index.html)

```php
// Returns the current competitive price of a product, based on ASIN.
$client->GetCompetitivePricingForASIN($asin_array = []);

// Returns the feed processing report and the Content-MD5 header.
$client->GetFeedSubmissionResult($FeedSubmissionId);

// Returns pricing information for the lowest-price active offer listings for up to 20 products, based on ASIN.
$client->GetLowestOfferListingsForASIN($asin_array = [], $ItemCondition = null);

// Returns lowest priced offers for a single product, based on ASIN.
$client->GetLowestPricedOffersForASIN($asin, $ItemCondition = 'New');

// Returns a list of products and their attributes, based on a list of ASIN, GCID, SellerSKU, UPC, EAN, ISBN, and JAN values.
$client->GetMatchingProductForId($asin_array, $type = 'ASIN');

// Returns a list of products and their attributes, based on an open text based query
$client->ListMatchingProducts($query, $query_context_id = null);

// Returns pricing information for your own offer listings, based on ASIN.
$client->GetMyPriceForASIN($asin_array = [], $ItemCondition = null);

// Returns pricing information for your own offer listings, based on SKU.
$client->GetMyPriceForSKU($sku_array = [], $ItemCondition = null);

// Returns an order based on the AmazonOrderId values that you specify.
$client->GetOrder($AmazonOrderId);

// Returns the parent product categories that a product belongs to, based on ASIN.
$client->GetProductCategoriesForASIN($ASIN);

// Returns the parent product categories that a product belongs to, based on SellerSKU.
$client->GetProductCategoriesForSKU($SellerSKU);

// Get a report's content
$client->GetReport($ReportId);

// Returns a list of reports that were created in the previous 90 days.
$client->GetReportList($ReportTypeList = []);

// Get a report's processing status
$client->GetReportRequestStatus($ReportId);

// Get a list's inventory for Amazon's fulfillment
$client->ListInventorySupply($sku_array = []);

// Returns a list of marketplaces that the seller submitting the request can sell in, and a list of participations that include seller-specific information in that marketplace
$client->ListMarketplaceParticipations();

// Returns order items based on the AmazonOrderId that you specify.
$client->ListOrderItems($AmazonOrderId);

// Returns orders created or updated during a time frame that you specify.
$client->ListOrders($from, $allMarketplaces = false, $states = ['Unshipped', 'PartiallyShipped'], $FulfillmentChannel = 'MFN');

// Returns your active recommendations for a specific category or for all categories for a specific marketplace.
$client->ListRecommendations($RecommendationCategory = null);

// Creates a report request and submits the request to Amazon MWS.
$client->RequestReport($report, $StartDate = null, $EndDate = null);

// Uploads a feed for processing by Amazon MWS.
$client->SubmitFeed($FeedType, $feedContent, $debug = false);

// Call this method to get the raw feed instead of sending it
$client->debugNextFeed();

// Post to create or update a product (_POST_FLAT_FILE_LISTINGS_DATA_)
$client->postProduct($MWSProduct);

// Update a product's price
$client->updatePrice($array);

// Update a product's stock quantity
$client->updateStock($array);

// A method to quickly check if the supplied credentials are valid
$client->validateCredentials();
```
