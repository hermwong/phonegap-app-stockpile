1. Create using Split-Panel template
  - `phonegap create Stockpile com.phonegap.stockpile Stockpile --template https://github.com/phonegap/phonegap-template-vue-f7-split-panel`
  - `cd Stockpile`
  - `npm install`

2. Test it out
  - `npm run dev` (will open a browser to `http://localhost:8080`) – OR – `npm run build && phonegap serve` and open Chrome to `http://localhost:3000` (if viewed at `localhost` or `127.0.0.1`, this should actually install the service worker, etc and can be tested with Lighthouse).
  - `npm test` (run unit and end to end tests)

3. Tweaks to _**index.html**_
  - change `<title />` to "Stockpile"

4. Home.vue -> Search.vue
  - rename `Home.vue` to `Search.vue`
  - `<f7-page name="home">` to `<f7-page name="search">`
  - change the `<f7-nav-center />` to `{{ title }}`
  - change the `title` data property to "Search"
  - `name: 'Home',` to `name: 'Search',`
  - update `routes.js` to use "Search" instead of "Home" (three instances)
  - replace `<f7-block-title />` and `<f7-block-inner />`  with:
    ```html
    <form form method="GET" id="search-form" @submit.prevent="onSubmit">
      <f7-list>
      </f7-list>
    </form>
    ```
  - in the default export, add a `methods` object with a stub (for now) for the `onSubmit()` function:
    ```javascript
    export default {
      name: 'Search',
      data () {
        return {
          title: 'Search'
        };
      },
      methods: {
        onSubmit () {}
      }
    };
    ```
  - in _**LeftPanel.vue**_, change the title of the first link from "Home" to "Search"

5. Start adding search form
  - add the search input:
    ```html
    <f7-list-item>
      <f7-label floating v-if="isMaterial">Image search</f7-label>
      <f7-input type="search" name="q"
        placeholder="Image search" ref="searchInput"
        autocorrect="off" autocapitalize="off" />
    </f7-list-item>
    ```
  - a hidden field to set the required limit and add the button to submit the search:
    ```html
    <f7-block>
      <input type="hidden" name="limit" value="60" />
      <input type="submit" name="submit" class="hidden" value="Search" />
      <f7-button @click.prevent="onSubmit" big raised fill>
        Find Images
      </f7-button>
    </f7-block>
    ```
  - add a computed property for `isMaterial`:
    ```javascript
    computed: {
      isMaterial () {
        return window.isMaterial;
      }
    },
    ```

6. `onSubmit()`
  - replace the stub `onSubmit()` with this:
    ```javascript
    onSubmit () {
      const { searchInput, searchForm } = this.$refs;
      const { filter, limit, q } = this.$f7.formToJSON(searchForm);
      const { router } = this.$f7.mainView;
      const input = searchInput.$el.querySelector('input');

      input.blur();

      if (!q.trim()) {
        this.$f7.alert('Please enter a search term', 'Search Error');
        return;
      }
      router.loadPage(`/results/${filter || 'words'}/${limit}/${q}/search`);
    }
    ```
    - _((explain this function))_

7. Add a little hack to ensure search is always at the top of the history (explain?)
    ```javascript
    created () {
      this.$f7.mainView.history = ['/'];
    }
    ```

8. The global store
  - in _**main.js**_, add a global `store` object:
    ```javascript
    // Global store defaults
    window.store = {
      images: [],
      imagesById: {}
    };
    ```

9. _**About.vue**_ -> _**Results.vue**_
  - edit `routes.js` and replace the import for `About` with `import Results from './components/pages/Results';`
  - edit `routes.js` and replace the route for `/about/` with:
    ```javascript
    {
      path: '/results/:filter/:limit/:q',
      component: Results
    },
    ```
  - remove the "About" link from `LeftPanel.vue`
  - rename `About.vue` to `Results.vue`
  - change `name` property to "Results"
  - `title` data object returns "Results"
  - replace entire `<f7-navbar ...` block with `<f7-navbar title="Results" back-link="Back" sliding></f7-navbar>`
  - in the `<f7-block-title>`, replace `{{ title }}` with `{{ imagesReturned }}`
  - modify the return expression for `data ()` in the default export to return the global store from above:
    ```javascript
    data () {
      return {
        images: []
      };
    }
    ```
  - in the default export after `data ()`, add a `computed` property:
    ```javascript
    imagesReturned () {
      // build the string to display for the number of results
      const { q } = this;
      const { filter } = this.$route.params;
      // wait for something to be returned
      if (!this.images) {
        return '';
      }
      switch (filter) {
        case 'similar':
          return this.images.length
          ? `${this.totalReturned} similar results to ${q}`
          : '';
        case 'creator_id':
          const [ img ] = this.images;
          return this.images.length
            ? `${this.totalReturned} results for ${img.creator_name}`
            : '';
        default:
          return this.images.length
            ? `${this.totalReturned} results for "${q}"`
            : '';
      }
    }
    ```
    - _((explain this function))_
  - add a `methods` property to the default export with a stub for `fetchResults ()`:
    ```javascript
    methods: {
      fetchResults () {}
    }
    ```

10. The images grid, reinitialization, and infinite scroll
  - replace the `<f7-page ...` opening tag with:
    ```html
    <f7-page name="results" @page:reinit="onPageReinit"
      infinite-scroll @infinite="onInfiniteScroll"
      :infinite-scroll-preloader="false"
    >
    ```
  - add stubs for the two event handlers above to the methods property
    ```javascript
    onInfiniteScroll () {},
    onPageReinit () {}
    ```
  - add a `results` boolean to the `data()` return: `results: true`
  - just before the closing `</f7-page>` tag, add a block to handle when there are no results returned:
    ```html
    <f7-block v-if="!results">
      <p class="center">No results found.</p>
      <p class="center">Go back and try a different search?</p>
    </f7-block>
    ```
  - immediately after, add the two preloaders. one is for the initial loading before there are any results, the second is to show that more images are loading in the infinite scroll
    ```html
    <div class="initial-preloader">
      <f7-preloader :style="images.length ? 'display: none; animation: none' : ''" />
    </div>
    <div class="infinite-scroll-preloader">
      <f7-preloader :style="images.length ? '' : 'display: none; animation: none'" />
    </div>
    ```
  - just below the `<f7-block-title ...` add the actual images grid block:
    ```html
    <f7-block>
      <div class="grid">
        <div class="cell" v-for="image in images">
          <img
            @click="() => onImageClick(image.id)"
            :src="image.thumbnail_url"
         />
        </div>
      </div>
    </f7-block>
    ```
  - add the `onImageClick` method to route from an image to the Details page:
    ```javascript
    onImageClick (id) {
      // route to the details page
      const { mainView: { router } } = this.$f7;
      router.loadPage(`/results/details/${id}`);
    }
    ```
  - after the `<script>` tag, add the CSS for the images grid
    ```css
    <style scoped>
      /* default for phones / portrait */
      .cell img {
        display: block;
        width: 100%;
      }
      .grid {
        background: #fff;
        display: flex;
        flex-wrap: wrap;
        flex-direction: row;
      }
      .cell {
        background: #fcfcfc;
        box-sizing: border-box;
        margin: 4px;
        width: calc(33% - 8px);
      }
      /* tablets / landscape */
      @media screen and (min-width: 960px) {
        .cell {
          width: calc(25% - 8px);
        }
      }
      /* desktop */
      @media screen and (min-width: 1200px) {
        .cell {
          width: calc(20% - 8px);
        }
      }
    </style>
    ```
    - _((explain this CSS))_
  - `onPageReinit()` will handle when we come back from deep navigation and want to display the correct results. it basically refreshes the store with the data from this view. replace the stub with:
    ```javascript
    onPageReinit () {
      // load the data for this page back into the store
      Object.assign(window.store, {
        images: this.images,
        imagesById: this.imagesById,
        totalReturned: this.totalReturned
      });
    }
    ```
  - `onInfiniteScroll()` loads the next set of images based on the limit/offset. replace the stub with:
    ```javascript
    onInfiniteScroll () {
      const limit = parseInt(this.limit, 10); // better safe
      const offset = parseInt(this.offset, 10); // ...than sorry
      if (this.totalReturned === this.images.length) {
        return;
      }
      this.fetchResults(this.q, limit, this.filter, offset);
    }
    ```

11. `fetchResults () {}`
  - first add the `fetch` polyfill: `npm install whatwg-fetch --save` (for Android 4.x and iOS < 10)
  - then add an import for it at the top of _**main.js**_: `import 'whatwg-fetch';`
  - in `index.html`, replace the CSP with:
    ```html
    <meta http-equiv="Content-Security-Policy" content="default-src 'self' data: gap: https://ssl.gstatic.com https://api.spotify.com 'unsafe-eval' 'unsafe-inline' ws://*; style-src 'self' 'unsafe-inline'; media-src *; img-src * data:">
    ```
  - create a folder in `src` called `utils` and in that folder create a file called `config.js` with the following:
    ```javascript
    export const apiHeaders = {
      'x-api-key': '***************************', // replace with your api-key
      'X-Product': 'Stockpile/1.0.0'
    };
    ```
  - replace the `***************************` with your Adobe Stock API key
  - also in the `utils` folder, create a file called `stockAPI.js` with the following:
    ```javascript
    /* global fetch */

    import { apiHeaders } from './config';

    const apiBase = 'https://stock.adobe.io/Rest/Media/1/Search/Files';

    // function to format an array of columns into the query string needed
    export function formatResultColumns (columns) {
      if (columns.length < 1) return '';
      return `result_columns[]=${columns.join('&result_columns[]=')}`;
    }

    // function to format an object containing parameters into the query string needed
    export function formatSearchParameters (parameters) {
      return parameters
        .map(param => `search_parameters[${param.key}]=${param.val}`)
        .join('&');
    }

    // function to call the Adobe Stock API and return the results
    export default function fetchStockAPIJSON (options) {
      const { columns, parameters } = options;
      const resultColumns = formatResultColumns(columns);
      const searchParameters = formatSearchParameters(parameters);
      const apiURL = `${apiBase}?${resultColumns}&${searchParameters}`;
      const myInit = {
        method: 'GET',
        headers: new Headers(apiHeaders)
      };
      return new Promise((resolve, reject) => {
        fetch(apiURL, myInit)
          .then(response => response.json())
          .then((json) => {
            resolve(json);
          }).catch((ex) => {
            reject(ex);
          });
      });
    }
    ```
    - _((explain these functions))_
  - back in _**Results.vue**_, add the real `fetchResults ()` method:
    ```javascript
    fetchResults (q, limit, filter, offset = 0) {
      const columns = [
        'nb_results', 'id', 'title', 'thumbnail_url', 'thumbnail_500_url',
        'thumbnail_1000_url', 'content_type', 'creation_date',
        'creator_name', 'creator_id', 'category', 'description',
        'content_type', 'keywords', 'comp_url'
      ];
      const parameters = [
        { key: 'thumbnail_size', val: '160' },
        { key: 'limit', val: limit },
        { key: filter, val: q },
        { key: 'offset', val: offset }
      ];

      fetchStockAPIJSON({ columns, parameters })
        .then(json => {
          // remove preloader if no results returned
          //  either from the end of the pagination or no results
          if (json.nb_results === 0) {
            this.$$('.initial-preloader').remove();
            this.$$('.infinite-scroll-preloader').remove();
          }

          // set initial totalReturned
          //  only if nb_results is > existing totalReturned
          //  this is because sometimes nb_results is 0
          if (json.nb_results >= this.totalReturned) {
            this.totalReturned = json.nb_results;
          }

          // set results bool to true if we have results
          //  and false if we do not
          this.results = !!this.totalReturned;

          // merge the two arrays adding in the new results
          this.images = this.images.concat(json.files);

          // reduce the images array into an object referenced by id...
          const imagesById = this.images.reduce((a, b) => {
            const c = a;
            c[b.id] = b;
            return c;
          }, {});

          // ...then merge with existing imagesById
          this.imagesById = Object.assign({}, this.imagesById, imagesById);

          // update the store
          // merging new and existing data using Object.assign()
          window.store = Object.assign(window.store, {
            images: this.images,
            imagesById: this.imagesById,
            totalReturned: this.totalReturned
          });

          // set the new offset
          this.offset = offset + limit; // not working currently...see: issue #4

          // remove the preloader if we have all the results
          if (json.files.length === 0 || this.totalReturned <= limit) {
            this.$$('.infinite-scroll-preloader').remove();
          }
        }).catch(ex => {
          console.log('fetching failed', ex);
          this.$f7.addNotification({
            title: 'Error',
            message: 'Failed to fetch from Adobe Stock',
            hold: 3000
          });
          this.$$('.infinite-scroll-preloader').remove();
        });
    }
    ```
    - _((explain this function - will need to be broken down a bit))_
    - _((might even need a refactor))_

12. Fire `fetchResults ()` in a lifecycle hook
  - add a `mounted ()` lifecycle hook to the default export:
    ```javascript
    mounted () {
      const { params } = this.$route;
      // set some initial defaults
      params.offset = parseInt(params.offset, 10) || 0;
      params.limit = parseInt(params.limit, 10) || 60;
      params.images = [];
      params.totalReturned = 0;
      Object.assign(this, params);
      this.fetchResults(this.q, this.limit, this.filter, this.offset);
    }
    ```
    _((explain this function and lifecycle hooks))_

13. _**Another.vue**_ -> _**Details.vue**_
  - rename `Another.vue` to `Details.vue`
  - edit `routes.js` and replace the import for `Another` with `import Results from './components/pages/Details';`
  - edit `routes.js` and replace the route for `/about/another/` with:
    ```javascript
    {
      path: '/results/details/:id',
      component: Results
    },
    ```
  - change `name` property to "Details"
  - replace the `data()` method with one returning the store:
    ```javascript
      data () {
        return store;
      }
      ```
  - replace entire `<f7-navbar ...` block with:
    ```html
    <f7-navbar :back-link="backLink" sliding>
      <f7-nav-center>
        Details
      </f7-nav-center>
      <f7-nav-right>
        <f7-link icon-f7="star_filled" @click="toggleFavorite"
          v-if="isFavorite"
        />
        <f7-link icon-f7="star" @click="toggleFavorite" v-else />
      </f7-nav-right>
    </f7-navbar>
    ```
  - in the default export after `data ()`, add a `computed` property for the backlink above:
    ```javascript
    computed: {
      backLink () {
        if (this.displayingFavorite) {
          return 'Favorites';
        } else {
          return 'Results';
        }
      }
    }
    ```
  - add another computed property for `displayingFavorite`:
    ```javascript
    displayingFavorite () {
      const { displayingFavorite = false } = this.$route.query;
      return !!displayingFavorite;
    }
    ```
  - add one more computed property for `isFavorite`:
    ```javascript
    isFavorite () {
      const filteredFavorites =
        this.favorites.filter(favorite => favorite.id.toString() === this.id);
      return !!filteredFavorites.length;
    }
    ```
  - in the default export after the `computed` property, add a `methods` property with a method for the `toggleFavorite` click handler (for now, just a stub, we'll come back to it):
    ```javascript
    methods: {
      toggleFavorite () {}
    }
    ```
  - add some computed properties for various links:
    ```javascript
    findMoreLink () {
      return `/results/similar/60/${this.item.id}/details`;
    },
    categoryLink () {
      return `/results/category/60/${this.item.category.id}/details`;
    },
    creatorLink () {
      return `/results/creator_id/60/${this.item.creator_id}/details`;
    }
    ```
  - replace the `<f7-block ...` with a card:
    ```html
    <f7-card>
    </f7-card>
    ```
  - inside the card, add the header:
    ```html
    <f7-card-header>
      <div class="img-container" :style="imgBackground()"
        @click="loadInPhotoBrowser"
      >
        <div class="img-container-inner" :style="imgBackground(500)"></div>
        <div class="caption">{{item.title}}</div>
      </div>
    </f7-card-header>
    ```
  - add a computed property for `id`:
    ```javascript
    id () {
      const { id } = this.$route.params;
      return id;
    }
    ```
  - add a computed property for the `item`:
    ```javascript
    item () {
      // Fallback default for when images* and favorites* are reset in
      //  the store
      if (this.displayingFavorite) {
        if (this.favoritesById && this.favoritesById[this.id]) {
          this.stockItem = Object.assign(
            {},
            this.stockItem,
            this.favoritesById[this.id]);
        }
        return this.stockItem;
      }
      if (this.imagesById && this.imagesById[this.id]) {
        this.stockItem = Object.assign(
          {},
          this.stockItem,
          this.imagesById[this.id]);
      }
      return this.stockItem;
    }
    ```
    - _((explain this function))_
  - add the `imgBackground()` method:
    ```javascript
    imgBackground (size = 0) {
      const url = size > 0 ? `thumbnail_${size}_url` : 'thumbnail_url';
      if (this.item[url]) this[url] = this.item[url];
      return `background-image: url(${this[url]})`;
    }
    ```
  - after the `<f7-card>`, add in the PhotoBrowser component to handle displaying the image
    ```html
    <f7-photo-browser
      ref="pb"
      type="page"
      :photos="photos"
      :lazyLoading="true"
      backLinkText="Details"
      :toolbar="false"
    ></f7-photo-browser>
    ```
  - add a computed property to return an array of ohotos for the Photo Browser with just our one image in it:
    ```javascript
    photos () {
      return [
        {
          url: this.item.thumbnail_1000_url,
          caption: this.item.title || ''
        }
      ];
    }
    ```
  - then add the `loadInPhotoBrowser()` method that will be called when the card header is clicked:
    ```javascript
    loadInPhotoBrowser () {
      this.$refs.pb.open();
    }
    ```
  - after the `<f7-card-header>` add in the `<f7-card-content>`:
    ```html
    <f7-card-content>
      <f7-list>
        <f7-list-item title="Category" :after="item.category.name"
          :link="categoryLink"
        ></f7-list-item>
        <f7-list-item
          title="Created by" :after="item.creator_name" :link="creatorLink"
        ></f7-list-item>
        <f7-list-item
          title="Creation date" :after="creationDate"></f7-list-item>
      </f7-list>
    </f7-card-content>
    ```
  - at the start of the `<script>` tag, add an import for moment.js: `import moment from 'moment';`
  - add in computed properties for `creationDate` using moment.js:
    ```javascript
    creationDate () {
      const created = moment(this.item.creation_date);
      return created.format('MMMM Do YYYY');
    }
    ```
  - lastly, add in the `<f7-card-footer>` after the `<f7-card-content>`:
    ```html
    <f7-card-footer>
      <f7-link :href="item.comp_url" external>Download Comp</f7-link>
      <f7-link :href="findMoreLink">Find Similar</f7-link>
    </f7-card-footer>
    ```
  - after the `<script>` tag, add a `<style>` tag like this:
    ```css
    <style scoped>
      .swiper {
        height: 300px;
      }
      .img-container {
        position: relative;
        width: 100%;
        height: 240px;
        max-height: 240px;
        overflow: hidden;
        margin: 0;
        background-size: cover;
        background-repeat: no-repeat;
        background-position: center center;
      }
      .img-container-inner {
        position:absolute;
        top: 50%;
        transform: translateY(-50%);
        -webkit-transform: translateY(-50%);
        width: 100%;
        height: 100%;
        background-size: cover;
        background-repeat: no-repeat;
        background-position: center center;
        z-index: 2;
      }
      .caption {
        position: absolute;
        bottom: 0;
        left: 0;
        right: 0;
        background: rgba(0,0,0,0.3);
        color: white;
        padding: 8px 16px;
        z-index: 3;
      }
    </style>
    ```
    - _((explain this css))_

14. `toggleFavorites()`
  - create a new file in `utils` called `favorites.js` and add a `toggleFavorite()` method:
    ```javascript
    export function toggleFavorite (favorite) {
      const alreadyAFavorite = store.favorites.filter(fave => fave.id === favorite.id);
      if (alreadyAFavorite.length > 0) {
        removeFavorite(favorite.id);
      } else {
        addFavorite(favorite);
      }
      saveFavoritesToLocalStorage();
    }
    ```
    - _((explain this function))_
  - add the `updateFavoritesById()` function to take the array of faves and store tham as an object keyed by their id:
    ```javascript
    function updateFavoritesById () {
      store.favoritesById = store.favorites.reduce((a, b) => {
        const c = a;
        c[b.id] = b;
        return c;
      }, {});
    }
    ```
  - add the `addFavorite()`, `removeFavorite()`, and `saveFavoritesToLocalStorage()` functions:
    ```javascript
    function addFavorite (favorite) {
      store.favorites.push(favorite);
      updateFavoritesById();
    }

    function removeFavorite (id) {
      store.favorites = store.favorites.filter(favorite => favorite.id !== id);
      updateFavoritesById();
    }

    function saveFavoritesToLocalStorage () {
      localStorage.setItem('favorites', JSON.stringify(store.favorites));
    }
    ```
    - _((explain these functions))_
  - back in **Details.vue**, add an import for the `favorites.js` file just under the import for moment.js: `import { toggleFavorite } from '../../utils/favorites';`
  - replace the `toggleFavorites()` stub with the following:
    ```javascript
    toggleFavorite () {
      const { mainView: { router } } = this.$f7;
      if (this.displayingFavorite) {
        // let the animation happen before removing the fave
        setTimeout(() => {
          toggleFavorite(this.item);
        }, 410);
        router.back();
      } else {
        toggleFavorite(this.item);
      }
    }
    ```

15. _**Services.vue**_ -> _**Favorites.vue**_
