<template>
  <div v-if="settings" class="restaurant-menu py-2.5" :class="settings.classes">
    <div>
      <!-- Restaurant Logo -->
      <div v-if="settings.logo && settings.logo.url" class="text-center mb-6">
        <nuxt-img :src="settings.logo.url" :alt="settings.logo.seo_tag || 'Restaurant Logo'" width="96" height="96"
          class="h-24 object-contain mx-auto" placeholder quality="80" format="webp" preload fetchpriority="high" />
      </div>

      <!-- Menu Title and Subtitle -->
      <div class="mb-8">
        <h2 v-if="menuTitle" class="menu-title mb-2 flex justify-center">
          {{ menuTitle }}
        </h2>
        <p v-if="menuSubtitle" class="menu-subtitle flex justify-center">
          {{ menuSubtitle }}
        </p>
      </div>

      <!-- Shopping Cart Icon - Only render the icon, not full cart -->
      <cart-icon :total-items="totalItems" @click="toggleCart" />

      <SocialLinks :links="socialMediaLinks" :show-whats-app="!!settings.showWhatsApp"
        :whatsapp-number="settings.whatsappNumber || ''"
        :whatsapp-message="settings.whatsappMessage && settings.whatsappMessage[lang] || ''" :cart="cart" :lang="lang"
        :type="'restaurant'" />

      <!-- Category View Mode -->
      <div v-if="isCategoryView" class="menu-content">
        <!-- Category Navigation Tabs - Optimized for performance -->
        <category-tabs :categories="sortedCategories" :active-category="activeCategory" :lang="lang"
          @select-category="setActiveCategory" />

        <!-- Active Category Description -->
        <div v-if="activeCategoryObj && activeCategoryObj.description && activeCategoryObj.description[lang]"
          class="mb-6 text-center">
          <p class="category-description">{{ activeCategoryObj.description[lang] }}</p>
        </div>

        <!-- Menu Items for Active Category - Using virtual list for large menus -->
        <menu-items-grid :items="getMenuItemsByCategory(activeCategory)" :currency-symbol="settings.currencySymbol"
          :lang="lang" @item-click="openItemModal" />
      </div>

      <!-- List View Mode (Original) - Optimized for performance -->
      <div v-else class="menu-content">
        <category-list :categories="sortedCategories" :get-items-by-category="getItemsByCategory"
          :currency-symbol="settings.currencySymbol" :lang="lang" @item-click="openItemModal" />
      </div>
    </div>

    <!-- Item Modal - Conditionally imported and rendered -->
    <item-modal v-if="showItemModal" :item="selectedItem" :currency-symbol="settings.currencySymbol" :lang="lang"
      :quantity="itemQuantity" :notes="itemNotes" @close="closeItemModal" @update-quantity="updateItemQuantity"
      @update-notes="updateItemNotes" @add-to-cart="addToCart" />

    <!-- Shopping Cart Sidebar - Conditionally imported and rendered -->
    <ShoppingCart v-if="showCart" :cart="cart" :currency-symbol="settings.currencySymbol"
      :tax-rate="settings.taxRate ? settings.taxRate / 100 : TAX_RATE"
      :enable-tax="settings.enableTax !== undefined ? settings.enableTax : true" :lang="lang" type="restaurant"
      :whatsapp-enabled="!!settings.showWhatsApp && !!settings.whatsappNumber"
      :whatsapp-number="settings.whatsappNumber || ''" @close="closeCart" @increment="incrementCartItem"
      @decrement="decrementCartItem" @remove="removeFromCart" @checkout="checkout" />
  </div>
</template>

<script>
// Import sub-components to allow code splitting and lazy loading
import CartIcon from '../../components/UnifiedMenu/CartIcon.vue';
import SocialLinks from '../../components/UnifiedMenu/SocialLinks.vue';


// Use dynamic imports for components not needed on initial render
const CategoryTabs = () => import('../../components/UnifiedMenu/CategoryTabs.vue');
const MenuItemsGrid = () => import('../../components/UnifiedMenu/ItemsGrid.vue');
const CategoryList = () => import('../../components/UnifiedMenu/CategoryList.vue');
const ItemModal = () => import('../../components/UnifiedMenu/ItemModal.vue');
const ShoppingCart = () => import('../../components/UnifiedMenu/ShoppingCart.vue');

// Utility functions for debouncing
const debounce = (fn, delay) => {
  let timeoutId;
  return function (...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
};

export default {
  name: 'RestaurantMenu',
  components: {
    CartIcon,
    SocialLinks,
    CategoryTabs,
    MenuItemsGrid,
    CategoryList,
    ItemModal,
    ShoppingCart
  },
  props: {
    section: {
      type: Object,
      default: () => { },
    },
    lang: {
      type: String,
      default: "en"
    },
    locales: {
      type: Array,
      default: () => [],
    },
    viewStructure: {
      settings: [
        {
          logo: 'image',
          menuTitle: {
            en: 'Our Menu',
            fr: 'Notre Menu'
          },
          menuSubtitle: {
            en: 'Discover our delicious options',
            fr: 'Découvrez nos délicieuses options'
          },
          categories: [
            {
              id: 'drinks-category-id',
              name: {
                en: 'Drinks',
                fr: 'Boissons'
              },
              description: {
                en: 'Category Description',
                fr: 'Description de la catégorie'
              },
              classes: '',
              icon: {
                media_id: '',
                url: '',
                seo_tag: ''
              }
            },
            {
              id: 'category-id',
              name: {
                en: 'Food',
                fr: 'Nourriture'
              },
              description: {
                en: '',
                fr: ''
              },
              classes: '',
              icon: {
                media_id: '',
                url: '',
                seo_tag: ''
              }
            },
            {
              id: 'category-id',
              name: {
                en: 'Salads',
                fr: 'Salades'
              },
              description: {
                en: '',
                fr: ''
              },
              classes: '',
              icon: {
                media_id: '',
                url: '',
                seo_tag: ''
              }
            }
          ],
          menuItems: [
            {
              id: 'item-id',
              categoryId: 'drinks-category-id',
              name: {
                en: 'Item Name',
                fr: 'Nom du plat'
              },
              description: {
                en: 'Item Description',
                fr: 'Description du plat'
              },
              price: 10.99,
              image: 'image',
              featured: true,
              classes: ''
            },
            {
              id: 'item-id',
              categoryId: 'drinks-category-id',
              name: {
                en: 'Item Name',
                fr: 'Nom du plat'
              },
              description: {
                en: 'Item Description',
                fr: 'Description du plat'
              },
              price: 10.99,
              image: 'image',
              featured: true,
              classes: ''
            },
            {
              id: 'item-id',
              categoryId: 'category-id',
              name: {
                en: 'Item Name',
                fr: 'Nom du plat'
              },
              description: {
                en: 'Item Description',
                fr: 'Description du plat'
              },
              price: 10.99,
              image: 'image',
              featured: false,
              classes: ''
            }
          ],
          currencySymbol: '$',
          classes: '',
          viewMode: 'list', // list or category
          enableTax: true,
          taxRate: 10.00
        }
      ]
    }
  },
  data() {
    return {
      cart: [],
      showCart: false,
      showItemModal: false,
      selectedItem: null,
      itemQuantity: 1,
      itemNotes: '',
      activeCategory: '',
      TAX_RATE: 0.10, // 10% tax rate
      isCartLoaded: false, // Track if cart has been loaded from storage
      socialIcons: {
        instagram: {
          classes: 'instagram',
          hoverClasses: 'instagram-hover'
        },
        facebook: {
          classes: 'facebook',
          hoverClasses: 'facebook-hover'
        },
        tiktok: {
          classes: 'tiktok',
          hoverClasses: 'tiktok-hover'
        },
        twitter: {
          classes: 'twitter',
          hoverClasses: 'twitter-hover'
        },
        youtube: {
          classes: 'youtube',
          hoverClasses: 'youtube-hover'
        }
      }
    };
  },
  computed: {
    menuTitle() {
      if (this.settings && this.settings.menuTitle && this.settings.menuTitle[this.lang]) {
        return this.settings.menuTitle[this.lang];
      }
      return '';
    },

    menuSubtitle() {
      if (this.settings && this.settings.menuSubtitle && this.settings.menuSubtitle[this.lang]) {
        return this.settings.menuSubtitle[this.lang];
      }
      return '';
    },
    settings() {
      if (!this.section?.settings) return {};
      // Always use the first element of the settings array
      return Array.isArray(this.section.settings) ? this.section.settings[0] : this.section.settings;
    },
    socialMediaLinks() {
      // Default to empty array if no social media
      if (!this.settings || !this.settings.socialMedia) return [];

      // Convert settings.socialMedia to an array of link objects
      return Object.entries(this.settings.socialMedia || {})
        .filter(([platform, url]) => {
          // Filter out empty URLs and WhatsApp
          return platform !== 'whatsapp' &&
            url &&
            typeof url === 'string' &&
            url.trim() !== '';
        })
        .map(([platform, url]) => {
          // Get the icon classes
          const classes = (this.socialIcons[platform] || {}).classes || '';
          const hoverClasses = (this.socialIcons[platform] || {}).hoverClasses || '';

          return {
            type: platform,
            url,
            classes,
            hoverClasses
          };
        });
    },
    sortedCategories() {
      if (!this.settings || !this.settings.categories) return [];

      // Return categories in their original order without sorting by order property
      return [...this.settings.categories];
    },
    isCategoryView() {
      return this.settings && this.settings.viewMode === 'category';
    },
    activeCategoryObj() {
      if (!this.activeCategory) return null;
      return this.settings.categories.find(cat => cat.id === this.activeCategory);
    },
    totalItems() {
      return this.cart.reduce((total, item) => total + item.quantity, 0);
    },
    cartSubtotal() {
      return this.cart.reduce((total, item) => total + (item.price * item.quantity), 0);
    },
    cartTax() {
      return this.cartSubtotal * this.TAX_RATE;
    },
    cartTotal() {
      return this.cartSubtotal + this.cartTax;
    }
  },
  watch: {
    // Watch for settings changes
    settings: {
      handler() {
        // Re-initialize when settings change
        this.initializeActiveCategory();
        this.initializeMenuTitles();
      },
      deep: true
    },

    // Watch for section settings changes directly
    "section.settings": {
      handler() {
        // Re-initialize when settings change
        this.initializeActiveCategory();
        this.initializeMenuTitles();
      },
      deep: true
    },

    // Watch for language changes
    lang() {
      this.initializeMenuTitles();
    }
  },
  created() {
    // Set initial active category if in category view mode
    this.initializeActiveCategory();

    // Initialize menu titles
    this.initializeMenuTitles();
    // The key fix: Check if settings exists before trying to access its properties
    if (this.settings) {
      // Initialize socialMedia if it doesn't exist
      if (!this.settings.socialMedia) {
        // Use Vue's $set to ensure the property is reactive
        this.$set(this.settings, 'socialMedia', {
          instagram: '',
          facebook: '',
          tiktok: '',
          twitter: '',
          youtube: ''
        });
      }

      // Initialize WhatsApp settings if they don't exist
      if (typeof this.settings.showWhatsApp === 'undefined') {
        this.$set(this.settings, 'showWhatsApp', false);
      }

      if (!this.settings.whatsappNumber) {
        this.$set(this.settings, 'whatsappNumber', '');
      }

      if (!this.settings.whatsappMessage) {
        this.$set(this.settings, 'whatsappMessage', {});
      }

      // Initialize whatsappMessage for the current language if not set
      if (this.settings.whatsappMessage && !this.settings.whatsappMessage[this.lang]) {
        this.$set(this.settings.whatsappMessage, this.lang, 'Hello! I would like to reserve a table.');
      }
    }
  },
  mounted() {
    // Defer non-critical initialization
    this.$nextTick(() => {
      // Load cart using idle callback if available, or defer with setTimeout
      if ('requestIdleCallback' in window) {
        window.requestIdleCallback(this.loadCartFromStorage);
      } else {
        setTimeout(this.loadCartFromStorage, 200);
      }
    });

    // Use event delegation for keyboard events
    document.addEventListener('keydown', this.handleKeyEvents);
  },
  beforeDestroy() {
    // Clean up event listeners
    document.removeEventListener('keydown', this.handleKeyEvents);
  },
  methods: {
    initializeMenuTitles() {
      if (this.settings) {
        // Initialize menuTitle if it doesn't exist
        if (!this.settings.menuTitle) {
          this.$set(this.settings, 'menuTitle', {});
        }

        // Initialize menuTitle for the current language
        if (!this.settings.menuTitle[this.lang]) {
          this.$set(this.settings.menuTitle, this.lang, '');
        }

        // Initialize menuSubtitle if it doesn't exist
        if (!this.settings.menuSubtitle) {
          this.$set(this.settings, 'menuSubtitle', {});
        }

        // Initialize menuSubtitle for the current language
        if (!this.settings.menuSubtitle[this.lang]) {
          this.$set(this.settings.menuSubtitle, this.lang, '');
        }
      }
    },
    // Load cart from storage (deferred)
    loadCartFromStorage() {
      if (this.isCartLoaded) return;

      // Check if we're in a browser environment first
      if (typeof window !== 'undefined') {
        const savedCart = localStorage.getItem('restaurantMenuCart');
        if (savedCart) {
          try {
            this.cart = JSON.parse(savedCart);
            this.isCartLoaded = true;
          } catch { }
        }
      }
    },
    initializeActiveCategory() {
      if (this.isCategoryView && this.sortedCategories.length > 0 && !this.activeCategory) {
        this.activeCategory = this.sortedCategories[0].id;
      }
    },
    // Optimized method to get items by category
    getItemsByCategory(categoryId) {
      if (!this.settings.menuItems) return [];
      return this.settings.menuItems.filter(item => item.categoryId === categoryId);
    },
    // Alias for consistency
    getMenuItemsByCategory(categoryId) {
      return this.getItemsByCategory(categoryId);
    },
    // Set active category (for category view)
    setActiveCategory(categoryId) {
      this.activeCategory = categoryId;
    },
    // Format price with better performance
    // Item modal methods
    openItemModal(item) {
      this.selectedItem = item;
      this.itemQuantity = 1;
      this.itemNotes = '';
      this.showItemModal = true;
      document.body.classList.add('overflow-hidden');
    },
    closeItemModal() {
      this.showItemModal = false;
      document.body.classList.remove('overflow-hidden');
    },
    updateItemQuantity(quantity) {
      this.itemQuantity = quantity;
    },
    updateItemNotes(notes) {
      this.itemNotes = notes;
    },
    // Cart toggle methods
    toggleCart() {
      // Make sure cart is loaded before showing
      if (!this.isCartLoaded) {
        this.loadCartFromStorage();
      }

      this.showCart = !this.showCart;
      if (this.showCart) {
        document.body.classList.add('overflow-hidden');
      } else {
        document.body.classList.remove('overflow-hidden');
      }
    },
    closeCart() {
      this.showCart = false;
      document.body.classList.remove('overflow-hidden');
    },
    // Efficient event handling with delegation
    handleKeyEvents(e) {
      if (e.key === 'Escape') {
        if (this.showItemModal) this.closeItemModal();
        if (this.showCart) this.closeCart();
      }
    },
    // Cart operations
    addToCart() {
      if (!this.selectedItem) return;

      const newItem = {
        id: this.selectedItem.id,
        name: this.selectedItem.name,
        price: this.selectedItem.price,
        quantity: this.itemQuantity,
        notes: this.itemNotes
      };

      // Close modal first for better perceived performance
      this.closeItemModal();

      // Then update cart in next tick to avoid blocking the UI
      this.$nextTick(() => {
        // Check if item already exists in cart
        const existingItemIndex = this.cart.findIndex(item =>
          item.id === newItem.id && item.notes === newItem.notes
        );

        if (existingItemIndex !== -1) {
          // Update existing item quantity
          this.cart[existingItemIndex].quantity += newItem.quantity;
        } else {
          // Add new item to cart
          this.cart.push(newItem);
        }

        // Save cart to localStorage with debounce
        this.debouncedSaveCart();

        // Show cart after a small delay for better UX
        setTimeout(() => {
          this.showCart = true;
        }, 100);
      });
    },
    removeFromCart(index) {
      this.cart.splice(index, 1);
      this.debouncedSaveCart();
    },
    incrementCartItem(index) {
      this.cart[index].quantity++;
      this.debouncedSaveCart();
    },
    decrementCartItem(index) {
      if (this.cart[index].quantity > 1) {
        this.cart[index].quantity--;
        this.debouncedSaveCart();
      }
    },
    // Debounced method to avoid excessive localStorage writes
    debouncedSaveCart: debounce(function () {
      localStorage.setItem('restaurantMenuCart', JSON.stringify(this.cart));
    }, 300),
    checkout() {
      // Here you would typically integrate with a payment processor
      // For now, we'll just show an alert
      alert(this.$t('RestaurantMenu.orderPlaced') || 'Your order has been placed!');
      this.cart = [];
      this.debouncedSaveCart();
      this.closeCart();
    }
  }
};
</script>
