# Shopify Recently Viewed Products Slider (OS 2.0)

A high-performance, lightweight, and dependency-free **Recently Viewed Products Carousel** designed specifically for Shopify Online Store 2.0 themes. 

Unlike third-party Shopify apps that inject render-blocking scripts, track personal shopper data, and charge monthly recurring fees, this solution runs entirely client-side using browser `localStorage` and Shopify's native Ajax API.

---

## 🚀 Key Benefits

### 1. Zero Monthly App Costs & Code Cleanliness
* **No recurring subscriptions:** Saves on monthly Shopify app charges for basic storefront functionality.
* **No app bloat:** Eliminates heavy external SDKs, remote tracking scripts, and residual orphaned code that typically slows down storefronts after app uninstalls.

### 2. Maximum Storefront Performance & Core Web Vitals
* **Zero External Dependencies:** Built with pure HTML5, vanilla ES6 JavaScript, and native CSS. It does not require jQuery, Swiper.js, or Slick Slider.
* **Zero Cumulative Layout Shift (CLS):** The container defaults to `display: none` and only unhides once product cards are fetched and appended. If a visitor has no viewing history, zero empty layout blocks or flashes occur.
* **Hardware-Accelerated Mobile UX:** Uses native CSS Scroll Snap (`scroll-snap-type: x mandatory; -webkit-overflow-scrolling: touch;`), delivering buttery-smooth 60fps swipe gestures on iOS and Android devices without JavaScript overhead.

### 3. Universal Theme Compatibility
* **Theme-Agnostic Architecture:** Does not rely on Dawn-specific snippets (`card-product.liquid`) or CSS stylesheets. It works out of the box on free themes (Dawn, Sense, Craft, Spotlight) as well as premium custom themes.
* **Multi-Currency Support:** Automatically formats prices according to the store's active currency using `window.Shopify.currency.active`.

### 4. Smart Display & Filtering Logic
* **Automatic PDP Filtering:** Excludes the item currently being viewed so shoppers never see the product they are actively inspecting inside its own "Recently Viewed" slider.
* **Deduplication:** Revisiting an item reorders it to position 1 without storing duplicate entries.
* **Multi-Page Flexibility:** Works seamlessly on both single Product Detail Pages (PDPs) and the Homepage.

---

## 🛠️ Step-by-Step Implementation Guide

Follow these steps to integrate the tracking system and responsive slider into your Shopify theme.

### Step 1: Add the Global Tracking Script

The tracker records product handles to `localStorage` across your catalog. Adding it to `theme.liquid` ensures products are recorded even if a specific product template does not have the visible slider enabled.

1. From your Shopify Admin, navigate to **Online Store > Themes**.
2. Click the **three dots (`...`)** next to your active theme and select **Edit code**.
3. In the file navigator under **Layout**, open `theme.liquid`.
4. Scroll to the bottom and locate the closing `</body>` tag.
5. Paste the following snippet directly **above** the `</body>` tag:

```liquid
{%- if template.name == 'product' -%}
  <script>
    (function () {
      try {
        const STORAGE_KEY = 'shopify_recently_viewed';
        const currentHandle = {{ product.handle | json }};
        if (!currentHandle) return;

        let history = JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
        
        // Remove duplicate if already present, then insert at the beginning
        history = history.filter(function (handle) {
          return handle !== currentHandle;
        });
        history.unshift(currentHandle);

        // Retain the last 20 viewed products in local memory
        if (history.length > 20) history.pop();

        localStorage.setItem(STORAGE_KEY, JSON.stringify(history));
      } catch (e) {
        console.warn('Could not record recently viewed item:', e);
      }
    })();
  </script>
{%- endif -%}

```

6. Click **Save**.

---

### Step 2: Create the Standalone Section File

This section handles the dynamic card retrieval via Shopify's `/products/{handle}.js` endpoint and manages the responsive carousel track.

1. In the theme code editor, scroll to the **Sections** folder.
2. Click **Add a new section**.
3. Select **Liquid** as the type and name the file `recently-viewed.liquid`.
4. Replace all placeholder code with the following:

```liquid
<div
  class="rv-section-container"
  data-section-id="{{ section.id }}"
  data-max-products="{{ section.settings.products_to_show }}"
  {% if template.name == 'product' %}
    data-current-product-handle="{{ product.handle }}"
  {% endif %}
  style="display: none;"
>
  <div class="rv-wrapper">
    <div class="rv-header">
      {% if section.settings.heading != blank %}
        <h2 class="rv-heading">{{ section.settings.heading | escape }}</h2>
      {% endif %}

      <div class="rv-slider-arrows">
        <button
          type="button"
          class="rv-arrow rv-arrow-prev"
          id="rv-prev-{{ section.id }}"
          aria-label="Previous products"
        >
          &#10094;
        </button>
        <button
          type="button"
          class="rv-arrow rv-arrow-next"
          id="rv-next-{{ section.id }}"
          aria-label="Next products"
        >
          &#10095;
        </button>
      </div>
    </div>

    <div
      class="rv-slider-track"
      id="rv-track-{{ section.id }}"
      style="--rv-items-desktop: {{ section.settings.columns_desktop }}; --rv-items-mobile: {{ section.settings.columns_mobile }};"
    >
      {%- comment -%} Product cards are dynamically injected here {%- endcomment -%}
    </div>
  </div>
</div>

<style>
  .rv-section-container {
    width: 100%;
    padding: 40px 20px;
    box-sizing: border-box;
    overflow: hidden;
  }
  .rv-wrapper {
    max-width: 1200px;
    margin: 0 auto;
    position: relative;
  }
  .rv-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 20px;
  }
  .rv-heading {
    margin: 0;
    font-size: 1.5rem;
    font-weight: 600;
  }
  .rv-slider-arrows {
    display: flex;
    gap: 8px;
  }
  .rv-arrow {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 36px;
    height: 36px;
    border-radius: 50%;
    border: 1px solid #e0e0e0;
    background: #ffffff;
    color: #111111;
    cursor: pointer;
    font-size: 14px;
    transition: background-color 0.2s, border-color 0.2s;
    user-select: none;
  }
  .rv-arrow:hover {
    background: #f4f4f4;
    border-color: #bbb;
  }

  /* Responsive Touch Slider Track */
  .rv-slider-track {
    display: flex;
    gap: 16px;
    overflow-x: auto;
    scroll-snap-type: x mandatory;
    scroll-behavior: smooth;
    -webkit-overflow-scrolling: touch;
    scrollbar-width: none;
    padding-bottom: 8px;
  }
  .rv-slider-track::-webkit-scrollbar {
    display: none;
  }

  /* Card Dimension & Flex Layout */
  .rv-card {
    flex: 0 0 calc((100% - (var(--rv-items-mobile, 1.3) - 1) * 16px) / var(--rv-items-mobile, 1.3));
    scroll-snap-align: start;
    display: flex;
    flex-direction: column;
    text-decoration: none;
    color: inherit;
    border-radius: 8px;
    overflow: hidden;
    background: #ffffff;
  }
  @media (min-width: 768px) {
    .rv-card {
      flex: 0 0 calc((100% - (var(--rv-items-desktop, 4) - 1) * 16px) / var(--rv-items-desktop, 4));
    }
  }

  .rv-image-wrapper {
    position: relative;
    width: 100%;
    padding-bottom: 100%;
    background: #f4f4f4;
    overflow: hidden;
    border-radius: 8px;
  }
  .rv-image-wrapper img {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    transition: transform 0.3s ease;
  }
  .rv-card:hover .rv-image-wrapper img {
    transform: scale(1.04);
  }
  .rv-info {
    padding: 12px 2px 4px;
    display: flex;
    flex-direction: column;
    gap: 4px;
  }
  .rv-vendor {
    font-size: 0.75rem;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: #666;
  }
  .rv-title {
    font-size: 0.95rem;
    line-height: 1.3;
    margin: 0;
    overflow: hidden;
    text-overflow: ellipsis;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
  }
  .rv-price {
    font-size: 0.95rem;
    font-weight: 600;
    display: flex;
    gap: 8px;
    align-items: baseline;
    margin-top: 2px;
  }
  .rv-compare-price {
    font-size: 0.85rem;
    text-decoration: line-through;
    color: #888;
    font-weight: normal;
  }
</style>

<script>
  (function () {
    const STORAGE_KEY = 'shopify_recently_viewed';
    const sectionContainer = document.querySelector('[data-section-id="{{ section.id }}"]');
    if (!sectionContainer) return;

    const currentHandle = sectionContainer.dataset.currentProductHandle || '';
    const maxDisplay = parseInt(sectionContainer.dataset.maxProducts, 10) || 8;
    const track = document.getElementById('rv-track-{{ section.id }}');
    const prevBtn = document.getElementById('rv-prev-{{ section.id }}');
    const nextBtn = document.getElementById('rv-next-{{ section.id }}');

    let history = [];
    try {
      history = JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
    } catch (e) {
      console.warn('Unable to access localStorage', e);
      return;
    }

    // Exclude current product if on PDP; keep all if on Homepage
    const handlesToFetch = history
      .filter(function (handle) {
        return handle !== currentHandle;
      })
      .slice(0, maxDisplay);

    if (handlesToFetch.length === 0) return;

    function formatMoney(cents) {
      const activeCurrency =
        window.Shopify && window.Shopify.currency && window.Shopify.currency.active
          ? window.Shopify.currency.active
          : 'USD';
      return (cents / 100).toLocaleString(undefined, {
        style: 'currency',
        currency: activeCurrency,
      });
    }

    const placeholderImg =
      'data:image/gif;base64,R0lGODlhAQABAIAAAMLCwgAAACH5BAAAAAAALAAAAAABAAEAAAICRAEAOw==';

    Promise.all(
      handlesToFetch.map(function (handle) {
        return fetch(window.Shopify.routes.root + 'products/' + handle + '.js')
          .then(function (res) {
            return res.ok ? res.json() : null;
          })
          .catch(function () {
            return null;
          });
      })
    ).then(function (products) {
      const validProducts = products.filter(function (p) {
        return p !== null;
      });

      if (validProducts.length === 0) return;

      const cardsHtml = validProducts
        .map(function (product) {
          const featuredImage = product.featured_image ? product.featured_image : placeholderImg;
          const hasDiscount = product.compare_at_price > product.price;
          const cleanTitle = product.title.replace(/"/g, '&quot;');

          let vendorHtml = '';
          {% if section.settings.show_vendor %}
            if (product.vendor) {
              vendorHtml = '<span class="rv-vendor">' + product.vendor + '</span>';
            }
          {% endif %}

          let discountHtml = '';
          if (hasDiscount) {
            discountHtml =
              '<span class="rv-compare-price">' +
              formatMoney(product.compare_at_price) +
              '</span>';
          }

          return (
            '<a href="' + product.url + '" class="rv-card">' +
              '<div class="rv-image-wrapper">' +
                '<img src="' + featuredImage + '" alt="' + cleanTitle + '" loading="lazy" width="300" height="300" />' +
              '</div>' +
              '<div class="rv-info">' +
                vendorHtml +
                '<h3 class="rv-title">' + cleanTitle + '</h3>' +
                '<div class="rv-price">' +
                  '<span>' + formatMoney(product.price) + '</span>' +
                  discountHtml +
                '</div>' +
              '</div>' +
            '</a>'
          );
        })
        .join('');

      track.innerHTML = cardsHtml;
      sectionContainer.style.display = 'block';

      // Desktop Arrow Scroll Controls
      if (prevBtn && nextBtn) {
        prevBtn.addEventListener('click', function () {
          const card = track.querySelector('.rv-card');
          const step = card ? card.offsetWidth + 16 : 280;
          track.scrollBy({ left: -step, behavior: 'smooth' });
        });

        nextBtn.addEventListener('click', function () {
          const card = track.querySelector('.rv-card');
          const step = card ? card.offsetWidth + 16 : 280;
          track.scrollBy({ left: step, behavior: 'smooth' });
        });
      }
    });
  })();
</script>

{% schema %}
{
  "name": "Recently Viewed",
  "tag": "section",
  "class": "section",
  "settings": [
    {
      "type": "text",
      "id": "heading",
      "default": "Recently Viewed",
      "label": "Heading"
    },
    {
      "type": "range",
      "id": "products_to_show",
      "min": 4,
      "max": 12,
      "step": 1,
      "default": 8,
      "label": "Maximum products to show"
    },
    {
      "type": "range",
      "id": "columns_desktop",
      "min": 2,
      "max": 5,
      "step": 1,
      "default": 4,
      "label": "Visible cards on desktop"
    },
    {
      "type": "select",
      "id": "columns_mobile",
      "options": [
        { "value": "1.2", "label": "1 Card (Peek next)" },
        { "value": "2.2", "label": "2 Cards (Peek next)" }
      ],
      "default": "2.2",
      "label": "Visible cards on mobile"
    },
    {
      "type": "checkbox",
      "id": "show_vendor",
      "default": false,
      "label": "Show vendor"
    }
  ],
  "presets": [
    {
      "name": "Recently Viewed"
    }
  ]
}
{% endschema %}

```

5. Click **Save**.

---

### Step 3: Add and Customize via Theme Editor

1. In your Shopify admin, go to **Online Store > Themes** and click **Customize**.
2. Select the template where you want the slider to appear:
* **Product template:** From the top-center dropdown, select **Products > Default product**.
* **Homepage:** Leave the dropdown set to **Home page**.


3. In the left panel, click **Add section** and select **Recently Viewed**.
4. Drag and position the section (typically directly below product recommendations or above the footer).
5. In the right-hand settings panel, adjust:
* **Heading:** Edit section title (default: *Recently Viewed*).
* **Maximum products to show:** Select between 4 and 12 products.
* **Visible cards on desktop:** Choose between 2 and 5 visible columns.
* **Visible cards on mobile:** Choose between `1.2` or `2.2` cards (partial card peeking encourages horizontal swiping).
* **Show vendor:** Toggle brand/vendor display above titles.


6. Click **Save**.

---

## 🧪 Testing Your Installation

1. Open your storefront in an **Incognito / Private Window**.
2. Visit 3 or 4 different product pages to allow the script to store their handles.
3. Scroll down on the 4th product page or navigate back to the Homepage.
4. Verify that:
* The recently viewed carousel renders your visited items in exact chronological order.
* The active product is filtered out on PDPs.
* Desktop arrow buttons smoothly paginate the cards.
* Mobile devices support native swipe gesture navigation.



---

## 📄 License

This project is open-source and released under the [MIT License](https://www.google.com/search?q=LICENSE).

```

```
