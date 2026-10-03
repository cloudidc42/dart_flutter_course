# Part 44: GraphQL
## ขั้นตอนที่ 1641-1680

---

## 🎯 เป้าหมายของ Part นี้

- GraphQL client setup
- Queries & Mutations
- Subscriptions (real-time)
- Cache management
- Code generation

---

## ขั้นตอนที่ 1641: Setup

```yaml
# pubspec.yaml
dependencies:
  graphql_flutter: ^5.2.0
  gql: ^1.0.1

dev_dependencies:
  artemis: ^7.12.0
  build_runner: ^2.4.9
```

---

## ขั้นตอนที่ 1642: GraphQL Client Setup

```dart
import 'package:flutter/material.dart';
import 'package:graphql_flutter/graphql_flutter.dart';

// ─── GraphQL Client ───
class GraphQLConfig {
  static HttpLink httpLink = HttpLink('https://api.example.com/graphql');

  static AuthLink authLink = AuthLink(
    getToken: () async {
      String? token = await SecureStorageService().getAuthToken();
      return token != null ? 'Bearer $token' : null;
    },
  );

  static WebSocketLink wsLink = WebSocketLink(
    'wss://api.example.com/graphql',
    config: const SocketClientConfig(
      autoReconnect: true,
      inactivityTimeout: Duration(seconds: 30),
    ),
  );

  static Link get link {
    Link authHttpLink = authLink.concat(httpLink);
    return Link.split(
      (request) => request.isSubscription,
      wsLink,
      authHttpLink,
    );
  }

  static ValueNotifier<GraphQLClient> get client {
    return ValueNotifier(
      GraphQLClient(
        link: link,
        cache: GraphQLCache(store: InMemoryStore()),
        defaultPolicies: DefaultPolicies(
          query: Policies(fetch: FetchPolicy.cacheAndNetwork),
          mutate: Policies(fetch: FetchPolicy.networkOnly),
        ),
      ),
    );
  }
}

// ─── Wrap App ───
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return GraphQLProvider(
      client: GraphQLConfig.client,
      child: MaterialApp(
        title: 'GraphQL App',
        home: const ProductsPage(),
      ),
    );
  }
}

// ─── GraphQL Queries ───
class GraphQLQueries {
  static const String getProducts = r'''
    query GetProducts($limit: Int!, $offset: Int!) {
      products(limit: $limit, offset: $offset) {
        id
        name
        price
        description
        imageUrl
        category {
          id
          name
        }
        rating {
          average
          count
        }
      }
      productsAggregate {
        count
      }
    }
  ''';

  static const String getProductById = r'''
    query GetProduct($id: ID!) {
      product(id: $id) {
        id
        name
        price
        description
        imageUrl
        inStock
        variants {
          id
          name
          price
        }
        reviews {
          id
          rating
          comment
          user {
            name
            avatarUrl
          }
        }
      }
    }
  ''';

  static const String searchProducts = r'''
    query SearchProducts($query: String!, $categoryId: ID) {
      searchProducts(query: $query, categoryId: $categoryId) {
        id
        name
        price
        imageUrl
      }
    }
  ''';
}

class GraphQLMutations {
  static const String addToCart = r'''
    mutation AddToCart($productId: ID!, $quantity: Int!) {
      addToCart(productId: $productId, quantity: $quantity) {
        id
        items {
          productId
          quantity
          product {
            name
            price
          }
        }
        total
      }
    }
  ''';

  static const String createOrder = r'''
    mutation CreateOrder($cartId: ID!, $addressId: ID!) {
      createOrder(cartId: $cartId, addressId: $addressId) {
        id
        status
        total
        estimatedDelivery
      }
    }
  ''';
}

class GraphQLSubscriptions {
  static const String orderStatusUpdated = r'''
    subscription OnOrderStatusUpdated($orderId: ID!) {
      orderStatusUpdated(orderId: $orderId) {
        id
        status
        updatedAt
        message
      }
    }
  ''';

  static const String newMessage = r'''
    subscription OnNewMessage($chatId: ID!) {
      newMessage(chatId: $chatId) {
        id
        content
        sender {
          id
          name
        }
        timestamp
      }
    }
  ''';
}
```

---

## ขั้นตอนที่ 1643: Query Widget

```dart
import 'package:flutter/material.dart';
import 'package:graphql_flutter/graphql_flutter.dart';

// ─── Products Page กับ GraphQL ───
class ProductsPage extends StatelessWidget {
  const ProductsPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Products')),
      body: Query(
        options: QueryOptions(
          document: gql(GraphQLQueries.getProducts),
          variables: const {'limit': 20, 'offset': 0},
          pollInterval: const Duration(seconds: 30), // auto refetch
        ),
        builder: (result, {fetchMore, refetch}) {
          if (result.isLoading && result.data == null) {
            return const Center(child: CircularProgressIndicator());
          }

          if (result.hasException) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text(result.exception.toString()),
                  ElevatedButton(
                    onPressed: () => refetch!(),
                    child: const Text('Retry'),
                  ),
                ],
              ),
            );
          }

          List products = result.data?['products'] ?? [];
          int total = result.data?['productsAggregate']?['count'] ?? 0;

          return Column(
            children: [
              Padding(
                padding: const EdgeInsets.all(8),
                child: Text('$total products total'),
              ),
              Expanded(
                child: RefreshIndicator(
                  onRefresh: () async => refetch!(),
                  child: GridView.builder(
                    gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                      crossAxisCount: 2,
                      crossAxisSpacing: 8,
                      mainAxisSpacing: 8,
                      childAspectRatio: 0.8,
                    ),
                    itemCount: products.length,
                    itemBuilder: (context, index) {
                      Map product = products[index];
                      return ProductCard(product: product);
                    },
                  ),
                ),
              ),
            ],
          );
        },
      ),
    );
  }
}

class ProductCard extends StatelessWidget {
  final Map product;
  const ProductCard({super.key, required this.product});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Expanded(
            child: Image.network(
              product['imageUrl'] ?? '',
              fit: BoxFit.cover,
              width: double.infinity,
              errorBuilder: (context, _, __) => const Icon(Icons.image, size: 60),
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(8),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(product['name'] ?? '', style: const TextStyle(fontWeight: FontWeight.bold)),
                Text('฿${product['price']?.toString() ?? '0'}'),
                Row(
                  children: [
                    const Icon(Icons.star, color: Colors.amber, size: 14),
                    Text('${product['rating']?['average']?.toStringAsFixed(1) ?? '0'}'),
                  ],
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1644: Mutation Widget

```dart
import 'package:flutter/material.dart';
import 'package:graphql_flutter/graphql_flutter.dart';

class AddToCartButton extends StatelessWidget {
  final String productId;
  const AddToCartButton({super.key, required this.productId});

  @override
  Widget build(BuildContext context) {
    return Mutation(
      options: MutationOptions(
        document: gql(GraphQLMutations.addToCart),
        onCompleted: (data) {
          if (data != null) {
            ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(content: Text('เพิ่มลงตะกร้าแล้ว!')),
            );
          }
        },
        onError: (error) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text('Error: $error')),
          );
        },
        update: (cache, result) {
          // Update cache manually
          // cache.writeQuery(...)
        },
      ),
      builder: (runMutation, result) {
        return ElevatedButton(
          onPressed: result?.isLoading == true
              ? null
              : () => runMutation({'productId': productId, 'quantity': 1}),
          child: result?.isLoading == true
              ? const SizedBox(
                  width: 16,
                  height: 16,
                  child: CircularProgressIndicator(strokeWidth: 2, color: Colors.white),
                )
              : const Text('เพิ่มลงตะกร้า'),
        );
      },
    );
  }
}

// ─── Programmatic GraphQL (no widget) ───
class GraphQLService {
  final GraphQLClient _client;
  GraphQLService(this._client);

  Future<List<Map<String, dynamic>>> getProducts({
    int limit = 20,
    int offset = 0,
  }) async {
    QueryResult result = await _client.query(
      QueryOptions(
        document: gql(GraphQLQueries.getProducts),
        variables: {'limit': limit, 'offset': offset},
      ),
    );

    if (result.hasException) throw result.exception!;
    return List<Map<String, dynamic>>.from(result.data?['products'] ?? []);
  }

  Future<Map<String, dynamic>> addToCart(String productId, int quantity) async {
    QueryResult result = await _client.mutate(
      MutationOptions(
        document: gql(GraphQLMutations.addToCart),
        variables: {'productId': productId, 'quantity': quantity},
      ),
    );

    if (result.hasException) throw result.exception!;
    return Map<String, dynamic>.from(result.data?['addToCart'] ?? {});
  }
}
```

---

## ขั้นตอนที่ 1645: Subscription Widget

```dart
import 'package:flutter/material.dart';
import 'package:graphql_flutter/graphql_flutter.dart';

// ─── Real-time Order Status ───
class OrderStatusTracker extends StatelessWidget {
  final String orderId;
  const OrderStatusTracker({super.key, required this.orderId});

  @override
  Widget build(BuildContext context) {
    return Subscription(
      options: SubscriptionOptions(
        document: gql(GraphQLSubscriptions.orderStatusUpdated),
        variables: {'orderId': orderId},
      ),
      builder: (result) {
        if (result.isLoading) {
          return const Center(child: CircularProgressIndicator());
        }

        if (result.hasException) {
          return Text('Connection error: ${result.exception}');
        }

        Map? orderUpdate = result.data?['orderStatusUpdated'];

        return Card(
          child: Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                const Row(
                  children: [
                    Icon(Icons.local_shipping),
                    SizedBox(width: 8),
                    Text('Order Status', style: TextStyle(fontWeight: FontWeight.bold)),
                    Spacer(),
                    // Live indicator
                    Icon(Icons.circle, color: Colors.green, size: 10),
                    SizedBox(width: 4),
                    Text('LIVE', style: TextStyle(color: Colors.green, fontSize: 12)),
                  ],
                ),
                const SizedBox(height: 12),
                if (orderUpdate != null) ...[
                  _StatusRow(
                    label: 'Status',
                    value: orderUpdate['status'] ?? '',
                    color: _getStatusColor(orderUpdate['status']),
                  ),
                  _StatusRow(
                    label: 'Message',
                    value: orderUpdate['message'] ?? '',
                  ),
                  _StatusRow(
                    label: 'Updated',
                    value: orderUpdate['updatedAt'] ?? '',
                  ),
                ] else
                  const Text('Waiting for updates...'),
              ],
            ),
          ),
        );
      },
    );
  }

  Color _getStatusColor(String? status) {
    switch (status) {
      case 'PROCESSING': return Colors.blue;
      case 'SHIPPED': return Colors.purple;
      case 'DELIVERED': return Colors.green;
      case 'CANCELLED': return Colors.red;
      default: return Colors.grey;
    }
  }
}

class _StatusRow extends StatelessWidget {
  final String label;
  final String value;
  final Color? color;

  const _StatusRow({required this.label, required this.value, this.color});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        children: [
          SizedBox(
            width: 80,
            child: Text(label, style: const TextStyle(color: Colors.grey)),
          ),
          Text(
            value,
            style: TextStyle(fontWeight: FontWeight.w500, color: color),
          ),
        ],
      ),
    );
  }
}
```

---

**← [Part 43 - State Management Advanced](part-43-state-management-advanced.md)**

**ต่อไป: [Part 45 - Flutter DevTools & Profiling →](part-45-devtools-profiling.md)**
