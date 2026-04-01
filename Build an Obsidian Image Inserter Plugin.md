![Everything](https://images.unsplash.com/photo-1452860606245-08befc0ff44b?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=MnwzNjAwOTd8MHwxfHNlYXJjaHw2fHx0b29sfGVufDB8MHx8fDE2NjM0MDE1MTg&ixlib=rb-1.2.1&q=80&w=1080)
*Photo by [Jo Szczepanska](https://unsplash.com/@joszczepanska?utm_source=Obsidian%20Image%20Inserter%20Plugin&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=Obsidian%20Image%20Inserter%20Plugin&utm_medium=referral)*

# Why need a plugin

Recently I use Obsidian as my note taking app and try to build a second brain based on it.
Obsidian is a note taking application based on markdown files. Plugin system is an awesome feature in Obsidian.
When writing blogs with Obsidian, sometimes I need to insert a related but random image to my blog to make it not that boring. So I'm thinking it would be great if I can search and insert image from [Unsplash](https://unsplash.com) inside Obsidian editor without leaving the editor and open browser tab to get the image.

# How the plugin works

First of all, Unsplash has an API to fetch images and their infos. This allows us implement the features like search images and insert it.
Obsidian plugin is a small JS script that runs when user installed and enabled the plugin. [@marcusolsson](https://github.com/marcusolsson) provide a fantastic [document](https://marcus.se.net/obsidian-plugin-docs/) for developping Obsidian plugins. It helps a lot.
The image inserter plugin should add a command to Obsidian. When users run this command, there should have a modal that shows a search input and a list of images that can be select. After the image has been selected, it should insert the image into the editor with correct markdown format immediately.

![three brow](https://images.unsplash.com/photo-1501785888041-af3ef285b470?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=MnwzNjAwOTd8MHwxfHNlYXJjaHw2fHxsYW5kc2NhcGV8ZW58MHwwfHx8MTY2MjA5MDA4NQ&ixlib=rb-1.2.1&q=80&w=1080)
*Photo by [Pietro De Grandi](https://unsplash.com/@peter_mc_greats?utm_source=obsidian-image-inserter&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=obsidian-image-inserter&utm_medium=referral)*

# Let's do it

### 1. Create a new plugin repo

Obsidian provides a [sample plugin repo](https://github.com/obsidianmd/obsidian-sample-plugin) as an initial starter. There are several files that we care about.
- `main.ts` is the entry point of the plugin
- inside `manifest.json` defines plugin id, name, author etc.
- `styles.css` allows us to customize the styles take effects on the plugin
We should change the sample plugin related contents to our plugin contents like class name. And also remove the demonstration codes that we don't need.

### 2. Add a command on the onload callback
As our plugin class extends the `Plugin` class, it get several inherited methods. Let's start with `onload` method. This method will be called when the plugin is loaded.
As shown below, we load settings, add the setting tab, and then add a command called *Insert Unsplash Image*.
Let's talk about the `addCommand`. We can register several different kinds of callbacks in this function.
- *callback* is the normal one, it will just run when user called our command
- *checkCallback* will run twice, first time it runs to check should this action to be perform or not, second time, it really perform the action.
- *editorCallback*, this callback let us access the editor, and it have the editor and the view as arguments.
- *editorCheckCallback*, a mix of "editorCallback" and "checkCallback"

We should use "editorCallback" this time. Our logic is simple, it just open a modal after being called.

```Javascript
export default class InsertUnsplashImage extends Plugin {
  ...

  async onload() {
  	this.loadSettings()
  	this.addSettingTab(new SettingTab(this.app, this));

  	this.addCommand({
  		id: 'image-inserter-plugin',
  		name: 'Insert Unsplash Image',
  		editorCallback: (editor: Editor) => {
      	new ImagesModal(this.app, editor, this.settings).open();
  		}
  	});
  }

  ...
}
```

### 3. Build the ImageModal component
This is the main part of our plugin. It will extends from the Obsidian built-in component called `SuggestModal`. `SuggestModal` is used to let user input something and get a result list and select one from the result.
We overload three methods here.
- `getSuggestions` will be called when user input something
- `renderSuggestion` will be called on each items that `getSuggestions` returned
- `onChooseSuggestion` will be called when user selected an image

```javascript
export class ImagesModal extends SuggestModal<Image> {
  ...

  async getSuggestions(query: string): Promise<Image[]> {
    // The promise is used for debounce, just ignore it
    return new Promise(resolve => {
      if (this.timer) clearTimeout(this.timer)

      this.timer = setTimeout(async () => {
        // SearchImages will search images with the query string and Unsplash API
        // It returns the images result
        // The settings.orientation is the preferred orientation
        // User can set preferred orientation at the plugin's setting tab
        const res = await this.fetcher.searchImages(query, this.settings.orientation)
        resolve(res)
      }, 500)
    })
  }

  onChooseSuggestion(item: Image) {
    // This is required by Unsplash for their usage counter, just ignore it
    this.fetcher.touchDownloadLocation(item.downloadLocationUrl)
    const url = item.url

    // ReplaceSelection means it will replace the user's selection with the string we passed to it
    // The replacement string is a bit of complicated because it is required by Unsplash to show the image's author and a back link to Unsplash
    this.editor.replaceSelection(`![${item.desc.slice(0, 10)}](${url})\n*Photo by [${item.author.name}](https://unsplash.com/@${item.author.username}?${UTM}) on [Unsplash](https://unsplash.com/?${UTM})*\n`)
  }

  renderSuggestion(value: Image, el: HTMLElement) {
    // Just create a element for every image
    // Then append it to the container
    const container = el.createEl("div", { attr: { style: "display: flex;" } })
    container.appendChild(el.createEl("img", { attr: { src: value.thumb, alt: value.desc } }))
  }

  ...
}
```

### 4. About the image fetcher request
It's a simple http request to Unsplash's API with some queries like per page and orientation.
The only problem is that we're not allowed the user to request to Unsplash directly. We let the user make a request to my own server and it's a reverse proxy server built on nginx. The reason is that for Unsplash API, it needs the client id to make the request and it forbid use to leak the client id to any other users. So I built the proxy server and it will receive any request, then send the request to Unsplash API with client id attached. That will allow us keep our client id secret.

```javascript
async searchImages(query: string, orientation: string): Promise<Image[]> {
  const url = new URL("/search/photos", "https://insert-unsplash-image.cloudy9101.com/")
  url.searchParams.set("query", query)
  if (orientation != 'not_specified') {
    url.searchParams.set("orientation", orientation)
  }
  url.searchParams.set("per_page", "9")
  const res = await fetch(url)
  const data: Unsplash.RootObject = await res.json()
  return data.results.map(function(item) {
    return {
      desc: item.description || item.alt_description,
      thumb: item.urls.thumb,
      url: item.urls.regular,
      downloadLocationUrl: item.links.download_location,
      author: {
        name: item.user.name,
        username: item.user.username,
      },
    }
  })
}
```

The Nginx config. I hide the normal things like server name, listen port etc.
```nginx
server {
  ...

  location / {
    set $args $args&client_id=xxx;
    proxy_pass https://api.unsplash.com;
    proxy_set_header Host $proxy_host;
  }

  ...
}
```

![Balloons a](https://images.unsplash.com/photo-1597132055361-ae81bab8f657?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=MnwzNjAwOTd8MHwxfHNlYXJjaHw3fHxjZWxlYnJhdGV8ZW58MHwwfHx8MTY2MzQwMTQyOQ&ixlib=rb-1.2.1&q=80&w=1080)
*Photo by [Felipe Santana](https://unsplash.com/@felipesantana?utm_source=Obsidian%20Image%20Inserter%20Plugin&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=Obsidian%20Image%20Inserter%20Plugin&utm_medium=referral)*


# Publish Plugin
Finally, we did it. Our simple plugin is finished. It's time to publish it to Obsidian community to share with the world.
There's the [document](https://marcus.se.net/obsidian-plugin-docs/publishing/submit-your-plugin) about publishing.
First, we need to build our plugin with `npm run build` and also update the manifest.json with correct values.
Second, we should push our repository to Github.
Then, create a release from Github with the three required files `main.js`, `manifest.json`,`styles.css`.
At the end, create a PR for [obsidian-releases](https://github.com/obsidianmd/obsidian-releases) to publish our plugin to Obsidian Plugins site.
After some review-fix-approve process, our PR will be merged. Then celebrate it.