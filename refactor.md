1.  src/pages/HomePage.jsx 

Add  useSearchParams  to your React Router import:

import { useSearchParams } from "react-router-dom";

Replace the local  searchTerm  state:

const [searchTerm, setSearchTerm] = useState("");

with URL-based state:

const [searchParams, setSearchParams] = useSearchParams();
const searchTerm = searchParams.get("search") ?? "";

Update  handleClearFilters :

function handleClearFilters() {
  setSearchParams((current) => {
    const next = new URLSearchParams(current);
    next.delete("search");
    return next;
  });
  setSelectedCategory("all");
}

In the  <ProductFilters />  call, remove the  searchTerm  and  onSearchChange  props. Keep its category-related props.

2.  src/components/ProductFilters.jsx 

Replace the file contents with:

function ProductFilters({
  categories,
  selectedCategory,
  onCategoryChange,
}) {
  return (
    <div className="products-filters">
      <label>
        <span className="visually-hidden">Filter by category</span>
        <select value={selectedCategory} onChange={onCategoryChange}>
          <option value="all">All</option>
          {categories.map((category) => (
            <option key={category} value={category}>
              {category.replaceAll("-", " ")}
            </option>
          ))}
        </select>
      </label>
    </div>
  );
}

export default ProductFilters;

3.  src/components/Navbar.jsx 

Add  useNavigate  and  useSearchParams  to the React Router import. Inside  Navbar , add:

const navigate = useNavigate();
const [searchParams] = useSearchParams();
const searchTerm = searchParams.get("search") ?? "";

function handleSearchChange(event) {
  const value = event.target.value;

  navigate(
    {
      pathname: "/",
      search: value ? `?search=${encodeURIComponent(value)}` : "",
    },
    { replace: true },
  );
}

Add this input between the brand link and the navigation links:

<label className="navbar-search">
  <span className="visually-hidden">Search products</span>
  <input
    type="search"
    placeholder="Search products..."
    value={searchTerm}
    onChange={handleSearchChange}
  />
</label>

In  src/styles/Navbar.css , add:

.navbar-search {
  flex: 1;
  margin: 0 2rem;
  max-width: 28rem;
}

.navbar-search input {
  box-sizing: border-box;
  font: inherit;
  min-height: 2.5rem;
  padding: 0.5rem 0.75rem;
  width: 100%;
}

.navbar-search input:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: 2px;
}

@media (max-width: 600px) {
  .navbar {
    flex-wrap: wrap;
    gap: 0.75rem;
  }

  .navbar-search {
    flex-basis: 100%;
    margin: 0;
    max-width: none;
    order: 3;
  }
}






1. Replace  src/store/CartContext.jsx 

import { createContext, useContext, useMemo, useState, useEffect } from "react";

const CartContext = createContext(null);
const CART_STORAGE_KEY = "shop.cart.v1";
const FALLBACK_STOCK_LIMIT = 99;

function getStockLimit(stock) {
  return Number.isFinite(stock)
    ? Math.max(0, Math.floor(stock))
    : FALLBACK_STOCK_LIMIT;
}

function loadCart() {
  try {
    const savedCart = window.localStorage.getItem(CART_STORAGE_KEY);
    if (!savedCart) return [];

    const parsedCart = JSON.parse(savedCart);
    if (!Array.isArray(parsedCart)) return [];

    return parsedCart
      .filter(
        (item) =>
          item &&
          item.id != null &&
          typeof item.title === "string" &&
          Number.isFinite(item.price) &&
          item.price >= 0 &&
          Number.isInteger(item.quantity) &&
          item.quantity > 0 &&
          getStockLimit(item.stock) > 0,
      )
      .map((item) => ({
        ...item,
        quantity: Math.min(item.quantity, getStockLimit(item.stock)),
      }));
  } catch (error) {
    console.error("Could not load the saved cart.", error);
    return [];
  }
}

export function CartProvider({ children }) {
  const [cart, setCart] = useState(loadCart);

  useEffect(() => {
    try {
      window.localStorage.setItem(CART_STORAGE_KEY, JSON.stringify(cart));
    } catch (error) {
      console.error("Could not save the cart.", error);
    }
  }, [cart]);

  function addToCart(product) {
    const stockLimit = getStockLimit(product.stock);
    if (stockLimit === 0) return;

    setCart((currentCart) => {
      const existingItem = currentCart.find((item) => item.id === product.id);

      if (existingItem) {
        if (existingItem.quantity >= stockLimit) return currentCart;

        return currentCart.map((item) =>
          item.id === product.id
            ? {
                ...item,
                stock: product.stock,
                quantity: item.quantity + 1,
              }
            : item,
        );
      }

      return [...currentCart, { ...product, quantity: 1 }];
    });
  }

  function removeFromCart(productId) {
    setCart((currentCart) =>
      currentCart.filter((item) => item.id !== productId),
    );
  }

  function increaseQuantity(productId) {
    setCart((currentCart) =>
      currentCart.map((item) => {
        if (
          item.id !== productId ||
          item.quantity >= getStockLimit(item.stock)
        ) {
          return item;
        }

        return { ...item, quantity: item.quantity + 1 };
      }),
    );
  }

  function decreaseQuantity(productId) {
    setCart((currentCart) =>
      currentCart.map((item) =>
        item.id === productId && item.quantity > 1
          ? { ...item, quantity: item.quantity - 1 }
          : item,
      ),
    );
  }

  const cartCount = useMemo(
    () => cart.reduce((total, item) => total + item.quantity, 0),
    [cart],
  );

  const cartTotal = useMemo(
    () => cart.reduce((total, item) => total + item.price * item.quantity, 0),
    [cart],
  );

  const value = {
    cart,
    cartCount,
    cartTotal,
    addToCart,
    removeFromCart,
    increaseQuantity,
    decreaseQuantity,
  };

  return (
    <CartContext.Provider value={value}>{children}</CartContext.Provider>
  );
}

export function useCart() {
  const context = useContext(CartContext);

  if (context === null) {
    throw new Error("useCart must be used within a CartProvider.");
  }

  return context;
}

The minus button now stops at 1; the separate Remove button removes the item. Products without a numeric stock value use a fallback limit of 99.

2. Replace  src/pages/CartPage.jsx 

import { Link } from "react-router-dom";
import { useCart } from "../store/CartContext";
import "../styles/CartPage.css";

const priceFormatter = new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD",
});

function CartPage() {
  const {
    cart,
    cartCount,
    cartTotal,
    removeFromCart,
    decreaseQuantity,
    increaseQuantity,
  } = useCart();

  if (cart.length === 0) {
    return (
      <section className="cart-empty">
        <h1>Your cart is empty</h1>
        <p>Browse the store and add something you like.</p>
        <Link className="cart-button cart-button-primary" to="/">
          Browse products
        </Link>
      </section>
    );
  }

  return (
    <section className="cart-page">
      <h1>Your cart</h1>

      <div className="cart-layout">
        <ul className="cart-list">
          {cart.map((item) => (
            <li className="cart-item" key={item.id}>
              <img
                className="cart-item-image"
                src={item.thumbnail}
                alt={item.title}
              />

              <div className="cart-item-details">
                <h2>{item.title}</h2>
                <p className="cart-muted">
                  {priceFormatter.format(item.price)} each
                </p>

                <div
                  className="cart-quantity"
                  aria-label={`Quantity for ${item.title}`}
                >
                  <button
                    type="button"
                    onClick={() => decreaseQuantity(item.id)}
                    disabled={item.quantity <= 1}
                    aria-label={`Decrease quantity of ${item.title}`}
                  >
                    −
                  </button>
                  <span aria-live="polite">{item.quantity}</span>
                  <button
                    type="button"
                    onClick={() => increaseQuantity(item.id)}
                    disabled={item.quantity >= getStockLimit(item.stock)}
                    aria-label={`Increase quantity of ${item.title}`}
                  >
                    +
                  </button>
                </div>
              </div>

              <div className="cart-item-actions">
                <strong>
                  {priceFormatter.format(item.price * item.quantity)}
                </strong>
                <button
                  className="cart-remove"
                  type="button"
                  onClick={() => removeFromCart(item.id)}
                  aria-label={`Remove ${item.title} from cart`}
                >
                  Remove
                </button>
              </div>
            </li>
          ))}
        </ul>

        <aside className="cart-summary" aria-label="Cart summary">
          <h2>Order summary</h2>
          <div className="cart-summary-row">
            <span>Items</span>
            <span>{cartCount}</span>
          </div>
          <div className="cart-summary-row cart-summary-total">
            <span>Total</span>
            <strong>{priceFormatter.format(cartTotal)}</strong>
          </div>
          <p className="cart-muted">Checkout will be added later.</p>
          <Link className="cart-button cart-button-primary" to="/">
            Continue shopping
          </Link>
        </aside>
      </div>
    </section>
  );
}

function getStockLimit(stock) {
  return Number.isFinite(stock) ? Math.max(0, Math.floor(stock)) : 99;
}

export default CartPage;

3. Create  src/styles/CartPage.css 

.cart-page h1 {
  margin: 0 0 1.5rem;
}

.cart-layout {
  align-items: start;
  display: grid;
  gap: 1.5rem;
  grid-template-columns: minmax(0, 1fr) 18rem;
}

.cart-list {
  list-style: none;
  margin: 0;
  padding: 0;
}

.cart-item {
  align-items: center;
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 0.75rem;
  display: grid;
  gap: 1rem;
  grid-template-columns: 6rem minmax(0, 1fr) auto;
  margin-bottom: 1rem;
  padding: 1rem;
}

.cart-item-image {
  aspect-ratio: 1;
  background: #f1f5f9;
  border-radius: 0.5rem;
  object-fit: contain;
  width: 100%;
}

.cart-item-details h2 {
  font-size: 1rem;
  margin: 0 0 0.4rem;
}

.cart-muted {
  color: #64748b;
  margin: 0.4rem 0 0.75rem;
}

.cart-quantity {
  align-items: center;
  display: inline-flex;
  gap: 0.75rem;
}

.cart-quantity button,
.cart-button,
.cart-remove {
  font: inherit;
}

.cart-quantity button {
  background: #fff;
  border: 1px solid #cbd5e1;
  border-radius: 0.4rem;
  cursor: pointer;
  min-height: 2.5rem;
  min-width: 2.5rem;
}

.cart-quantity button:disabled {
  cursor: not-allowed;
  opacity: 0.45;
}

.cart-item-actions {
  align-items: end;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.cart-remove {
  background: none;
  border: 0;
  color: #b42318;
  cursor: pointer;
  padding: 0.4rem 0;
  text-decoration: underline;
}

.cart-summary {
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 0.75rem;
  padding: 1.25rem;
}

.cart-summary h2 {
  font-size: 1.1rem;
  margin: 0 0 1rem;
}

.cart-summary-row {
  display: flex;
  justify-content: space-between;
  margin: 0.75rem 0;
}

.cart-summary-total {
  border-top: 1px solid #e2e8f0;
  padding-top: 1rem;
}

.cart-button {
  align-items: center;
  border-radius: 0.5rem;
  display: inline-flex;
  justify-content: center;
  min-height: 2.75rem;
  padding: 0.5rem 1rem;
  text-decoration: none;
}

.cart-button-primary {
  background: #1d4ed8;
  color: #fff;
  width: 100%;
}

.cart-button-primary:hover {
  background: #1e40af;
}

.cart-empty {
  align-items: center;
  display: flex;
  flex-direction: column;
  margin: 4rem auto;
  max-width: 28rem;
  text-align: center;
}

.cart-empty p {
  color: #64748b;
  margin-bottom: 1.5rem;
}

.cart-quantity button:focus-visible,
.cart-remove:focus-visible,
.cart-button:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: 3px;
}

@media (max-width: 760px) {
  .cart-layout {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 480px) {
  .cart-item {
    align-items: start;
    gap: 0.75rem;
    grid-template-columns: 4.5rem minmax(0, 1fr);
    padding: 0.75rem;
  }

  .cart-item-actions {
    align-items: center;
    flex-direction: row;
    grid-column: 2;
    justify-content: space-between;
  }
}

4. Update the cart count in  src/components/Navbar.jsx 

Change the hook usage and remove the local  cartCount  calculation:

const { cartCount } = useCart();

The existing link can stay as-is:

<NavLink to="/cart">Cart ({cartCount})</NavLink>

Then run  npm run lint  and  npm run build  from  D:\product\eCommerce_Frontend\@latest . Manually check add, increase up to stock, decrease, remove, empty cart, and refresh persistence.










carousel

1. Create  src/components/CategoryCarousel.jsx 

import { useEffect, useRef, useState } from "react";

function CategoryCarousel({
  categories,
  selectedCategory,
  onSelectCategory,
  totalCount,
}) {
  const trackRef = useRef(null);
  const [canScrollLeft, setCanScrollLeft] = useState(false);
  const [canScrollRight, setCanScrollRight] = useState(false);

  useEffect(() => {
    const track = trackRef.current;
    if (!track) return;

    function updateScrollButtons() {
      setCanScrollLeft(track.scrollLeft > 1);
      setCanScrollRight(
        track.scrollLeft + track.clientWidth < track.scrollWidth - 1,
      );
    }

    updateScrollButtons();
    track.addEventListener("scroll", updateScrollButtons, { passive: true });
    window.addEventListener("resize", updateScrollButtons);

    return () => {
      track.removeEventListener("scroll", updateScrollButtons);
      window.removeEventListener("resize", updateScrollButtons);
    };
  }, [categories]);

  function scrollTrack(direction) {
    trackRef.current?.scrollBy({
      left: direction * 220,
      behavior: "smooth",
    });
  }

  const categoryCards = [
    {
      name: "all",
      label: "All products",
      count: totalCount,
      image: null,
    },
    ...categories.map((category) => ({
      ...category,
      label: category.name.replaceAll("-", " "),
    })),
  ];

  return (
    <section className="category-carousel" aria-label="Filter by category">
      <div className="category-carousel-heading">
        <h2>Shop by category</h2>

        <div className="category-carousel-controls">
          <button
            type="button"
            onClick={() => scrollTrack(-1)}
            disabled={!canScrollLeft}
            aria-label="Scroll categories left"
          >
            ‹
          </button>
          <button
            type="button"
            onClick={() => scrollTrack(1)}
            disabled={!canScrollRight}
            aria-label="Scroll categories right"
          >
            ›
          </button>
        </div>
      </div>

      <ul className="category-carousel-track" ref={trackRef}>
        {categoryCards.map((category) => (
          <li className="category-carousel-item" key={category.name}>
            <button
              className="category-card"
              type="button"
              aria-pressed={selectedCategory === category.name}
              onClick={() => onSelectCategory(category.name)}
            >
              {category.image ? (
                <img
                  className="category-card-image"
                  src={category.image}
                  alt=""
                  loading="lazy"
                />
              ) : (
                <span
                  className="category-card-placeholder"
                  aria-hidden="true"
                />
              )}

              <span className="category-card-shade" aria-hidden="true" />

              <span className="category-card-content">
                <span className="category-card-name">{category.label}</span>
                <span className="category-card-count">
                  {category.count}{" "}
                  {category.count === 1 ? "product" : "products"}
                </span>
              </span>
            </button>
          </li>
        ))}
      </ul>
    </section>
  );
}

export default CategoryCarousel;

2. Replace  src/components/ProductFilters.jsx 

This keeps the existing search field but removes the category dropdown.

function ProductFilters({ searchTerm, onSearchChange }) {
  return (
    <div className="products-filters">
      <label>
        <span className="visually-hidden">Search products</span>
        <input
          type="search"
          placeholder="Search products..."
          value={searchTerm}
          onChange={onSearchChange}
        />
      </label>
    </div>
  );
}

export default ProductFilters;

3. Replace  src/pages/HomePage.jsx 

import { useEffect, useMemo, useState } from "react";
import { getProducts } from "../services/productService";
import "../styles/ProductsCatalog.css";
import CategoryCarousel from "../components/CategoryCarousel";
import ProductFilters from "../components/ProductFilters";
import ProductList from "../components/ProductList";
import ProductHeading from "../components/ProductHeading";

function HomePage() {
  const [products, setProducts] = useState([]);
  const [searchTerm, setSearchTerm] = useState("");
  const [selectedCategory, setSelectedCategory] = useState("all");
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState("");

  useEffect(() => {
    async function loadProducts() {
      try {
        setProducts(await getProducts());
      } catch (err) {
        setError(err.message || "Unable to load products.");
      } finally {
        setLoading(false);
      }
    }

    loadProducts();
  }, []);

  const categories = useMemo(() => {
    const categoriesByName = new Map();

    for (const product of products) {
      const existingCategory = categoriesByName.get(product.category);

      if (existingCategory) {
        existingCategory.count += 1;
      } else {
        categoriesByName.set(product.category, {
          name: product.category,
          count: 1,
          image: product.thumbnail,
        });
      }
    }

    return [...categoriesByName.values()].sort((a, b) =>
      a.name.localeCompare(b.name),
    );
  }, [products]);

  const filteredProducts = products.filter((product) => {
    const matchesSearch = product.title
      .toLowerCase()
      .includes(searchTerm.trim().toLowerCase());

    const matchesCategory =
      selectedCategory === "all" ||
      product.category === selectedCategory;

    return matchesSearch && matchesCategory;
  });

  function handleClearFilters() {
    setSearchTerm("");
    setSelectedCategory("all");
  }

  if (loading) {
    return <p role="status">Loading products...</p>;
  }

  if (error) {
    return <p role="alert">Error: {error}</p>;
  }

  return (
    <section className="products">
      <ProductHeading count={filteredProducts.length} />

      <CategoryCarousel
        categories={categories}
        selectedCategory={selectedCategory}
        onSelectCategory={setSelectedCategory}
        totalCount={products.length}
      />

      <ProductFilters
        searchTerm={searchTerm}
        onSearchChange={(event) => setSearchTerm(event.target.value)}
      />

      {filteredProducts.length > 0 ? (
        <ProductList products={filteredProducts} />
      ) : (
        <div className="empty-state">
          <p className="empty-results">No products match your filters.</p>
          <button type="button" onClick={handleClearFilters}>
            Clear filters
          </button>
        </div>
      )}
    </section>
  );
}

export default HomePage;

4. Update  src/styles/ProductsCatalog.css 

Keep your existing product-grid, heading, empty-state, and  .visually-hidden  styles. Replace the old  .products-filters  and select/input rules with these, then add the carousel styles:

.products-filters {
  display: flex;
  margin-bottom: 1.5rem;
}

.products-filters input {
  background: white;
  border: 1px solid #cbd5e1;
  border-radius: 0.5rem;
  box-sizing: border-box;
  font: inherit;
  min-height: 2.75rem;
  padding: 0.5rem 0.75rem;
  width: min(100%, 24rem);
}

.products-filters input:focus-visible {
  border-color: #2563eb;
  outline: 2px solid #2563eb;
  outline-offset: 2px;
}

.category-carousel {
  margin: 1.5rem 0;
}

.category-carousel-heading {
  align-items: center;
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.75rem;
}

.category-carousel-heading h2 {
  font-size: 1.15rem;
  margin: 0;
}

.category-carousel-controls {
  display: flex;
  gap: 0.5rem;
}

.category-carousel-controls button {
  background: white;
  border: 1px solid #cbd5e1;
  border-radius: 50%;
  cursor: pointer;
  font: inherit;
  font-size: 1.4rem;
  height: 2.5rem;
  line-height: 1;
  width: 2.5rem;
}

.category-carousel-controls button:disabled {
  cursor: not-allowed;
  opacity: 0.4;
}

.category-carousel-track {
  display: flex;
  gap: 0.75rem;
  list-style: none;
  margin: 0;
  overflow-x: auto;
  overscroll-behavior-x: contain;
  padding: 0.25rem 0.1rem 0.75rem;
  scroll-snap-type: x mandatory;
  scrollbar-width: none;
}

.category-carousel-track::-webkit-scrollbar {
  display: none;
}

.category-carousel-item {
  flex: 0 0 180px;
  scroll-snap-align: start;
}

.category-card {
  background: #334155;
  border: 2px solid transparent;
  border-radius: 0.75rem;
  color: white;
  cursor: pointer;
  display: block;
  height: 145px;
  overflow: hidden;
  padding: 0;
  position: relative;
  text-align: left;
  width: 100%;
}

.category-card[aria-pressed="true"] {
  border-color: #2563eb;
  box-shadow: 0 0 0 2px white inset;
}

.category-card-image,
.category-card-placeholder,
.category-card-shade {
  height: 100%;
  inset: 0;
  position: absolute;
  width: 100%;
}

.category-card-image {
  object-fit: cover;
  transition: transform 180ms ease;
}

.category-card:hover .category-card-image {
  transform: scale(1.05);
}

.category-card-placeholder {
  background: linear-gradient(135deg, #2563eb, #0f172a);
}

.category-card-shade {
  background: linear-gradient(
    to top,
    rgb(15 23 42 / 90%),
    rgb(15 23 42 / 5%) 80%
  );
}

.category-card-content {
  bottom: 0;
  display: grid;
  gap: 0.25rem;
  left: 0;
  padding: 0.75rem;
  position: absolute;
  right: 0;
}

.category-card-name {
  font-size: 1rem;
  font-weight: 700;
  text-transform: capitalize;
}

.category-card-count {
  font-size: 0.8rem;
}

.category-card:focus-visible,
.category-carousel-controls button:focus-visible {
  outline: 3px solid #2563eb;
  outline-offset: 3px;
}

@media (max-width: 560px) {
  .products-heading {
    align-items: start;
    gap: 0.5rem;
  }

  .products-filters input {
    width: 100%;
  }

  .category-carousel-item {
    flex-basis: 155px;
  }

  .category-card {
    height: 130px;
  }
}

How it works

•  HomePage  owns the selected category.  selectedCategory  starts as  "all" . Clicking a carousel card calls  setSelectedCategory  with that card’s category name.
• The category cards are derived from the loaded products.  useMemo  groups products by  product.category , counts each group, and uses the first product’s thumbnail as the category image. The “All products” card uses the complete loaded product count.
• Filtering still happens in one place. The existing product filter checks both  searchTerm  and  selectedCategory ; choosing a carousel card updates the category condition, while the search input continues to work.
• The carousel scrolls horizontally. CSS provides horizontal overflow and snap points. The arrow buttons move the track, and their disabled state updates as the user scrolls or resizes the window.
• Accessibility is built into the controls. Cards are buttons,  aria-pressed  exposes which category is selected, decorative images have empty alt text, and keyboard focus has a visible outline.

One limitation in the current project:  getProducts()  requests only 12 products, so the carousel’s categories and counts represent those 12. The carousel will work with the current data; showing every category in the API would require changing how products are fetched.

After applying the changes, run  npm run lint  and  npm run build . Test selecting a category, returning to “All products,” and scrolling the carousel on both a narrow and wide screen.









I am doing web dev now, but LLMs fascinate me, they seduced me to get closer to them want to make something out of it, that's why i get new dream of becoming an AI engineer. road is tough, i have no prior knowledge and experience in this field, but i am ready spend 7-8 months to learn, implement and deploy it. But i also have lingering worry deep inside my heart that these same LLMs, agents are being ruthless and getting better and better everyday. that is the reason lots of companies are firng and doing mass LayOffs and i heard fresher do not get job easily for an role of AI engineer i don't want to spend my 7-8 months for nothing. I am ready to do mathematics, algebra, stats, DSA, learn AI related stuff(not prompting , but ML, DL, NLP, tf-tdf and lot more interesting stuff, of course i won't deep dive in everyone , that is not the scope of 7-8 months abd it does not possible for my level to explore all in deep dive, i wiill mainly focus on 2 domain like NLP or GenAI , and deep dive in them . As i learn domains might also change , but learning won't change, my dedication for it, wont change, there will be time where i won't understand a thing, but i won't waver , i won't succumb , i won't give up. I will stand again, persevere and do it again, as even if i fail to get a job that pays handsomely. I will still try my best, Final goal is to get a position at the level of Staff engineer at company like OpenAI, i know first i need to drink the many potions of experience and level up myself. But starting small is my strategy for now and climb that ladder of Absolute greateness , i break sometime, i get weird thoughts , but somehow i also get healed by spending some time alone and thinking over thtough it and get pass to the mental hurdles. i have big dreams and up until now i was not putting effort ,wasting time but now i have learnt world doesn't stop for anybody, world does not not give a damn about pity and playing victim card. world rewards hard workers, recognizes people who rose from nothing and still do better everyday. It is an Univrsal law, But only worry that i mentioned was , what if after 1 year jobs like AI engineer also gets swept by LLMs and agentic AI. and i am starting my career and i don't have no turning back. Because it is big decision. 







