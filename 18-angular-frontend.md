# 🅰️ Module 18: Angular Frontend Integration

> All examples integrate with the **ShopEase** backend REST APIs.

---

## 1. What is Angular?

| Aspect | Details |
|---|---|
| **Type** | Client-side framework |
| **Developed by** | Google |
| **Language** | TypeScript |
| **Architecture** | Component-based, SPA (Single Page Application) |

---

## 2. Angular Setup

```bash
# Step 1: Install Node.js (https://nodejs.org)
node -v

# Step 2: Install TypeScript
npm install -g typescript
tsc -v

# Step 3: Install Angular CLI
npm install @angular/cli -g
ng v

# Step 4: Create ShopEase Frontend
ng new shopease-frontend
cd shopease-frontend
ng serve --open    # Opens http://localhost:4200
```

---

## 3. Angular Building Blocks

| Block | Purpose | ShopEase Example |
|---|---|---|
| **Component** | Small portion of UI | `ProductListComponent` |
| **Template** | HTML view | `product-list.component.html` |
| **Data Binding** | Connect component ↔ template | `{{ product.name }}` |
| **Directives** | Manipulate DOM | `*ngFor`, `*ngIf` |
| **Pipes** | Transform data | `{{ price | currency:'INR' }}` |
| **Services** | Business logic & API calls | `ProductService` |

### Component Structure

```
product-list/
├── product-list.component.ts      ← Logic
├── product-list.component.html    ← Template
├── product-list.component.css     ← Styles
```

---

## 4. ShopEase Angular Integration

### Product Model

```typescript
export class Product {
    constructor(
        public id: number,
        public name: string,
        public description: string,
        public price: number,
        public imageUrl: string,
        public category: string,
        public unitsInStock: number
    ) {}
}
```

### Product Service (API Calls)

```typescript
@Injectable({ providedIn: 'root' })
export class ProductService {

    private apiUrl = 'http://localhost:8080/api/products';

    constructor(private http: HttpClient) {}

    getAllProducts(): Observable<Product[]> {
        return this.http.get<Product[]>(this.apiUrl);
    }

    getProductById(id: number): Observable<Product> {
        return this.http.get<Product>(`${this.apiUrl}/${id}`);
    }

    searchProducts(keyword: string): Observable<Product[]> {
        return this.http.get<Product[]>(`${this.apiUrl}/search?q=${keyword}`);
    }
}
```

### Product List Component

```typescript
@Component({
    selector: 'app-product-list',
    templateUrl: './product-list.component.html',
    styleUrls: ['./product-list.component.css']
})
export class ProductListComponent implements OnInit {

    products: Product[] = [];

    constructor(private productService: ProductService) {}

    ngOnInit(): void {
        this.productService.getAllProducts().subscribe(data => {
            this.products = data;
        });
    }
}
```

### Product List Template

```html
<div class="product-grid">
    <div *ngFor="let product of products" class="product-card">
        <img [src]="product.imageUrl" [alt]="product.name">
        <h3>{{ product.name }}</h3>
        <p>{{ product.description }}</p>
        <span class="price">{{ product.price | currency:'INR' }}</span>
        <button (click)="addToCart(product)">Add to Cart</button>
    </div>
</div>
```

### Enable CORS on Backend

```java
// ShopEase Product Controller — allow Angular frontend
@RestController
@RequestMapping("/api/products")
@CrossOrigin(origins = "http://localhost:4200")
public class ProductController { ... }
```

---

## 5. Routing

```typescript
// app.routes.ts
export const routes: Routes = [
    { path: 'products', component: ProductListComponent },
    { path: 'products/:id', component: ProductDetailsComponent },
    { path: 'category/:id', component: ProductListComponent },
    { path: 'search/:keyword', component: ProductListComponent },
    { path: 'cart', component: CartDetailsComponent },
    { path: '', redirectTo: '/products', pathMatch: 'full' }
];
```

---

*← [17 — AWS Cloud](./17-aws-cloud.md) | [19 — JMeter Performance →](./19-jmeter-performance.md)*
