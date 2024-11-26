<template>
  <div id="app">
    <header>
      <h1>Lessons Shop</h1>
      <div class="cart">
        <span>🛒 Cart: {{ cartItemCount }} items</span>
      </div>
    </header>

    <div class="controls">
      <input 
        type="text" 
        v-model="searchQuery" 
        placeholder="Search lessons..." 
        class="search-bar"
      />
      <select v-model="sortOption" class="sort-select">
        <option disabled value="">Sort by</option>
        <option value="subject">Subject (A-Z)</option>
        <option value="location">Location (A-Z)</option>
        <option value="priceAsc">Price (Low to High)</option>
        <option value="stockAsc">Availability (Low to High)</option>
      </select>
    </div>

    <div v-if="isLoading" class="loading">
      <p>Loading lessons...</p>
    </div>
    <div v-else-if="products.length === 0" class="no-products">
      <p>No lessons available at the moment.</p>
    </div>

    <div v-if="showProductList && products.length > 0" class="product-list">
      <h2>Products</h2>
      <div v-for="product in sortedAndFilteredProducts" :key="product._id" class="product">
        <h3>{{ product.subject }}</h3>
        <img :src="product.imagePath" :alt="product.imageAlt" class="product-image" />
        <p>Location: {{ product.location }}</p>
        <p>Price: ${{ product.price }}</p>
        <p>Available: {{ product.stock }} 
          <span v-if="product.stock <= 2" class="low-stock">Low stock!</span>
        </p>
        <div class="cart-buttons">
          <button :disabled="product.stock === 0" @click="addToCart(product)">
            Add to Cart
          </button>
          <button :disabled="!isInCart(product)" @click="removeFromCart(product)">
            Remove from Cart
          </button>
        </div>
      </div>

      <button @click="goToCheckout" :disabled="cartItemCount === 0" class="checkout-btn">
        Go to Checkout
      </button>
    </div>

    <div v-if="showCheckoutPopup && !showUserForm" class="checkout-area">
      <h2>Checkout</h2>
      <div v-for="(item, index) in cart" :key="index" class="cart-item">
        <h3>{{ item.subject }}</h3>
        <img :src="item.imagePath" :alt="item.imageAlt" class="cart-image" />
        <p><strong>Price:</strong> ${{ item.price }}</p>
        <p><strong>Quantity:</strong> 
          <input type="number" v-model.number="item.quantity" min="1" :max="item.stock" @change="updateCart(index)" />
        </p>
        <p><strong>Subtotal:</strong> ${{ (item.price * item.quantity).toFixed(2) }}</p>
        <div class="cart-buttons">
          <button @click="removeFromCart(item, item.quantity)">Remove</button>
        </div>
      </div>

      <p><strong>Total Price:</strong> ${{ totalPrice.toFixed(2) }}</p>
      <button @click="goBackToProducts" class="back-btn">Back to Products</button>
      <button @click="showUserFormPopup" class="finalize-btn">
        Finalize Order
      </button>
    </div>

    <div v-if="showUserForm" class="user-form-overlay">
      <div class="user-form-container">
        <h2>Enter Your Details</h2>
        <form @submit.prevent="finalizeOrder">
          <label for="name">Name</label>
          <input type="text" id="name" v-model="user.name" required />
          <label for="phone">Phone</label>
          <input type="text" id="phone" v-model="user.phone" required />
          <label for="email">Email</label>
          <input type="email" id="email" v-model="user.email" required />
          <label for="address">Address</label>
          <input type="text" id="address" v-model="user.address" required />
          <button type="submit" :disabled="!isFormValid">Submit Order</button>
          <button type="button" @click="closeUserFormPopup">Cancel</button>
        </form>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      products: [],
      cart: [],
      user: { name: '', phone: '', email: '', address: '' },
      searchQuery: '',
      sortOption: '',
      showCheckoutPopup: false,
      showProductList: true,
      showUserForm: false,
      isLoading: true
    };
  },
  mounted() {
    this.fetchProducts(); // Fetch lessons on component mount
  },
  methods: {
    async fetchProducts() {
      this.isLoading = true;
      try {
        const response = await fetch('https://fullstack-express-9dbh.onrender.com/api/lessons');
        if (!response.ok) throw new Error(`Error ${response.status}: Unable to fetch products.`);
        const data = await response.json();
        this.products = Array.isArray(data) ? data : [];
      } catch (error) {
        console.error('Error fetching products:', error);
        alert('Failed to load products. Please try again.');
      } finally {
        this.isLoading = false;
      }
    },
    addToCart(product) {
      const existing = this.cart.find(item => item._id === product._id);
      if (existing) existing.quantity++;
      else this.cart.push({ ...product, quantity: 1 });
      product.stock--;
    },
    removeFromCart(product, quantity = 1) {
      const index = this.cart.findIndex(item => item._id === product._id);
      if (index >= 0) {
        const cartItem = this.cart[index];
        cartItem.quantity -= quantity;
        if (cartItem.quantity <= 0) this.cart.splice(index, 1);
        product.stock += quantity;
      }
    },
    goToCheckout() {
      this.showProductList = false;
      this.showCheckoutPopup = true;
    },
    goBackToProducts() {
      this.showProductList = true;
      this.showCheckoutPopup = false;
    },
    showUserFormPopup() {
      this.showUserForm = true;
    },
    closeUserFormPopup() {
      this.showUserForm = false;
    },
    async finalizeOrder() {
  const order = {
    name: this.user.name,
    phoneNumber: this.user.phone,
    email: this.user.email,
    address: this.user.address,
    lessonIDs: this.cart.map(item => item._id),
    numberOfSpaces: this.cart.map(item => item.quantity)
  };

  try {
    // Step 1: Submit user details to the 'users' collection (POST request)
    const userResponse = await fetch('https://fullstack-express-9dbh.onrender.com/api/users', {
      method: 'POST',
      headers: { 
        'Content-Type': 'application/json' 
      },
      body: JSON.stringify(this.user) // Send the user object as JSON
    });

    // Step 2: Handle response for user creation
    if (!userResponse.ok) {
      const errorText = await userResponse.text();
      throw new Error(`Failed to submit user details. Response: ${errorText}`);
    }

    // Step 3: Proceed with order submission if user creation is successful
    const orderResponse = await fetch('https://fullstack-express-9dbh.onrender.com/api/orders', {
      method: 'POST',
      headers: { 
        'Content-Type': 'application/json' 
      },
      body: JSON.stringify(order) // Send the order object as JSON
    });

    // Step 4: Handle response for order creation
    if (!orderResponse.ok) {
      const errorText = await orderResponse.text();
      throw new Error(`Failed to submit order. Response: ${errorText}`);
    }

    // Step 5: Update stock after successful order submission
    await this.updateStock();

    // Step 6: Clear cart and reset user data after successful order
    this.cart = [];
    this.user = { name: '', phone: '', email: '', address: '' };
    alert('Order placed successfully!');

    // Step 7: Go back to the products view
    this.goBackToProducts();
  } catch (error) {
    console.error('Error finalizing order:', error);
    alert(`Failed to finalize order: ${error.message}`);
  }
},
    async updateStock() {
      const stockUpdatePromises = this.cart.map(async (item) => {
        const updatedProduct = { ...item, stock: item.stock - item.quantity };
        const productResponse = await fetch(`https://fullstack-express-9dbh.onrender.com/api/lessons/${item._id}`, {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(updatedProduct)
        });
        if (!productResponse.ok) {
          throw new Error(`Failed to update stock for product ${item.subject}`);
        }
      });

      await Promise.all(stockUpdatePromises);
    },
    isInCart(product) {
      return this.cart.some(item => item._id === product._id);
    },
    isFormValid() {
      const nameRegex = /^[A-Za-z\s'-]+$/;
      const phoneRegex = /^[+]?[\d\s()-]+$/;
      return nameRegex.test(this.user.name) && phoneRegex.test(this.user.phone);
    }
  },
  computed: {
    cartItemCount() {
      return this.cart.reduce((sum, item) => sum + item.quantity, 0);
    },
    totalPrice() {
      return this.cart.reduce((sum, item) => sum + item.price * item.quantity, 0);
    },
    sortedAndFilteredProducts() {
      let filtered = this.products.filter(product =>
        product.subject.toLowerCase().includes(this.searchQuery.toLowerCase())
      );
      if (this.sortOption === 'subject') {
        filtered.sort((a, b) => a.subject.localeCompare(b.subject));
      } else if (this.sortOption === 'location') {
        filtered.sort((a, b) => a.location.localeCompare(b.location));
      } else if (this.sortOption === 'priceAsc') {
        filtered.sort((a, b) => a.price - b.price);
      } else if (this.sortOption === 'stockAsc') {
        filtered.sort((a, b) => a.stock - b.stock);
      }
      return filtered;
    }
  }
};
</script>

<style scoped>
/* General Styling */
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  background-color: #f9f9f9;
  color: #333;
  margin: 0;
  padding: 20px;
}

/* Header */
header {
  background-color: #4CAF50;
  color: white;
  padding: 15px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

header h1 {
  font-size: 1.8rem;
}

.cart {
  font-size: 1rem;
}

/* Controls */
.controls {
  display: flex;
  justify-content: center;
  margin: 20px 0;
}

.search-bar {
  padding: 10px;
  margin-right: 10px;
  width: 250px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.sort-select {
  padding: 10px;
  width: 200px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

/* Product List */
.product-list {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 20px;
}

.product {
  background-color: white;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  padding: 15px;
  width: 280px;
  text-align: center;
  transition: transform 0.2s, box-shadow 0.2s;
}

.product:hover {
  transform: translateY(-5px);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
}

.product-image {
  width: 100%;
  height: 150px;
  object-fit: cover;
  border-radius: 4px;
  margin-bottom: 10px;
}

.low-stock {
  color: red;
  font-weight: bold;
}

.cart-buttons {
  display: flex;
  justify-content: space-between;
  margin-top: 10px;
}

button {
  background-color: #4CAF50;
  color: white;
  padding: 10px 15px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:hover:not(:disabled) {
  background-color: #45a049;
}

button:disabled {
  background-color: #ddd;
  cursor: not-allowed;
}

.checkout-btn {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 1rem;
}

/* Cart in Checkout (Horizontal Layout) */
.checkout-area {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.cart-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: white;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  padding: 15px;
  margin-bottom: 20px;
  width: 90%;
  max-width: 800px;
}

.cart-image {
  width: 80px;
  height: 80px;
  object-fit: cover;
  border-radius: 4px;
  margin-right: 15px;
}

.cart-item-details {
  flex-grow: 1;
  text-align: left;
}

.cart-item-details p {
  margin: 5px 0;
}

.cart-buttons {
  margin-left: 10px;
}

.finalize-btn, .back-btn {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 1rem;
}

/* User Form */
.user-form-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.user-form-container {
  background: white;
  padding: 30px;
  border-radius: 8px;
  width: 300px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
  text-align: left;
}

input[type="text"], input[type="email"], input[type="number"] {
  padding: 10px;
  margin: 10px 0;
  width: 100%;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-sizing: border-box;
}

button[type="submit"] {
  width: 100%;
  font-size: 1rem;
  background-color: #4CAF50;
}

button[type="submit"]:disabled {
  background-color: #ddd;
}

/* Loading and Empty States */
.loading, .no-products {
  text-align: center;
  color: #555;
  font-size: 1.2rem;
  margin-top: 30px;
}
</style>
