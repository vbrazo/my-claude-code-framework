---
name: mobile-development
description: Cross-platform mobile UI — React Native and Flutter components, navigation, responsive layouts, and platform-specific code
---

# Mobile development

## A React Native list component

```tsx
import { View, Text, FlatList, StyleSheet, Platform } from "react-native";
import { SafeAreaView } from "react-native-safe-area-context";

interface Product {
  id: string;
  name: string;
  price: number;
  image: string;
}

function ProductList({ products }: { products: Product[] }) {
  return (
    <SafeAreaView style={styles.container}>
      <FlatList
        data={products}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => <ProductCard product={item} />}
        contentContainerStyle={styles.list}
        ItemSeparatorComponent={() => <View style={styles.separator} />}
        ListEmptyComponent={<EmptyState message="No products found" />}
        initialNumToRender={10}
        maxToRenderPerBatch={10}
        windowSize={5}
      />
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: "#fff",
  },
  list: {
    padding: 16,
  },
  separator: {
    height: 12,
  },
});
```

Render scrollable lists with `FlatList`, never a `ScrollView` over `.map()`, and tune `windowSize` and `maxToRenderPerBatch` once the list gets long.

## Navigation in React Native

```tsx
import { NavigationContainer } from "@react-navigation/native";
import { createNativeStackNavigator } from "@react-navigation/native-stack";
import { createBottomTabNavigator } from "@react-navigation/bottom-tabs";

type RootStackParams = {
  Tabs: undefined;
  ProductDetail: { productId: string };
  Cart: undefined;
};

const Stack = createNativeStackNavigator<RootStackParams>();
const Tab = createBottomTabNavigator();

function TabNavigator() {
  return (
    <Tab.Navigator screenOptions={{ headerShown: false }}>
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Search" component={SearchScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Tabs" component={TabNavigator} options={{ headerShown: false }} />
        <Stack.Screen name="ProductDetail" component={ProductDetailScreen} />
        <Stack.Screen name="Cart" component={CartScreen} options={{ presentation: "modal" }} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

## A Flutter widget

```dart
class ProductCard extends StatelessWidget {
  final Product product;
  final VoidCallback onTap;

  const ProductCard({
    super.key,
    required this.product,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Card(
        elevation: 2,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            ClipRRect(
              borderRadius: const BorderRadius.vertical(top: Radius.circular(8)),
              child: Image.network(
                product.imageUrl,
                height: 200,
                width: double.infinity,
                fit: BoxFit.cover,
                errorBuilder: (_, __, ___) => const Icon(Icons.broken_image, size: 64),
              ),
            ),
            Padding(
              padding: const EdgeInsets.all(12),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(product.name, style: Theme.of(context).textTheme.titleMedium),
                  const SizedBox(height: 4),
                  Text("\$${product.price.toStringAsFixed(2)}",
                      style: Theme.of(context).textTheme.bodyLarge),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

## Responsive layouts

```tsx
import { useWindowDimensions } from "react-native";

function useResponsive() {
  const { width } = useWindowDimensions();
  return {
    isPhone: width < 768,
    isTablet: width >= 768 && width < 1024,
    isDesktop: width >= 1024,
    columns: width < 768 ? 1 : width < 1024 ? 2 : 3,
  };
}

function ProductGrid({ products }: { products: Product[] }) {
  const { columns } = useResponsive();

  return (
    <FlatList
      data={products}
      numColumns={columns}
      key={columns}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => (
        <View style={{ flex: 1, maxWidth: `${100 / columns}%`, padding: 8 }}>
          <ProductCard product={item} />
        </View>
      )}
    />
  );
}
```

## Branching per platform

```tsx
import { Platform } from "react-native";

const styles = StyleSheet.create({
  shadow: Platform.select({
    ios: {
      shadowColor: "#000",
      shadowOffset: { width: 0, height: 2 },
      shadowOpacity: 0.1,
      shadowRadius: 4,
    },
    android: {
      elevation: 4,
    },
    default: {},
  }),
});
```

## What to avoid

- A `ScrollView` over `.map()` for dynamic lists — use `FlatList` or `SectionList`
- Dumping all state into a global store instead of colocating it with components
- Ignoring safe areas — notch, status bar, home indicator
- Rebuilding inline styles every render instead of `StyleSheet.create`
- Heavy computation on the JS thread — defer it with `InteractionManager`
- Brushing past platform UX conventions like the iOS back swipe or Android back button

## Before you ship

- [ ] Every scrollable list is a `FlatList` with a `keyExtractor`
- [ ] Navigation routes are typed with TypeScript params
- [ ] Safe-area insets are handled via `SafeAreaView`
- [ ] Styles come from `StyleSheet.create`, not inline objects
- [ ] Layouts adapt across phone, tablet, and desktop
- [ ] Platform-specific styling goes through `Platform.select`
- [ ] Images are cached and render error/loading states
- [ ] Heavy computation runs off the JS thread
