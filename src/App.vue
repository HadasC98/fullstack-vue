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
        // Submit user details to users collection
        const userResponse = await fetch('https://fullstack-express-9dbh.onrender.com/api/users', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(this.user)
        });

        if (!userResponse.ok) {
          const errorText = await userResponse.text();
          throw new Error(`Failed to submit user details. Response: ${errorText}`);
        }

        // Proceed with order submission if user creation is successful
        const orderResponse = await fetch('https://fullstack-express-9dbh.onrender.com/api/orders', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(order)
        });

        if (!orderResponse.ok) {
          throw new Error('Failed to submit order.');
        }

        // Handle stock update
        await this.updateStock();

        // Clear cart and reset user data
        this.cart = [];
        this.user = { name: '', phone: '', email: '', address: '' };
        alert('Order placed successfully!');

        // Go back to products view
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
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
}

.product-list {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
}

.product {
  border: 1px solid #ccc;
  padding: 10px;
  margin: 10px;
  width: 300px;
  text-align: center;
}

.product-image {
  width: 100px;
  height: 100px;
  object-fit: cover;
}

.low-stock {
  color: red;
  font-weight: bold;
}

.cart-buttons {
  display: flex;
  justify-content: space-between;
}

button {
  background-color: #4CAF50;
  color: white;
  padding: 10px 15px;
  border: none;
  cursor: pointer;
}

button:disabled {
  background-color: #ddd;
}

.checkout-btn {
  margin-top: 20px;
}

.checkout-area {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.cart-item {
  margin-bottom: 20px;
}

.cart-image {
  width: 80px;
  height: 80px;
}

.finalize-btn, .back-btn {
  margin-top: 20px;
}

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
}

input[type="text"], input[type="email"], input[type="number"] {
  padding: 10px;
  margin: 10px;
  width: 100%;
  border: 1px solid #ccc;
}
</style>
