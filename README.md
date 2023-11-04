# Leptos Image

[![Crates.io](https://img.shields.io/crates/v/leptos_image.svg)](https://crates.io/crates/leptos_image)
[![docs.rs](https://docs.rs/leptos_image/badge.svg)](https://docs.rs/leptos_image)

> Crafted with inspiration from Next.js

Images make a substantial impact on the size and performance of a website, so why not get them right?

Enter Leptos `<Image/>`, a component that enhances the standard HTML `<img>` element with automatic image optimization features.

- **Size Optimization**: Resize images, and use modern `.webp` format for optimal size and quality.
- **Low-Quality Image Placeholders (LQIP)**: With this feature, the Image component embeds SVGs, extracted from the original images, into the initial SSR HTML. This placeholder is shown while the optimized version is loading.
- **Faster Page Load**: Prioritize key images, such as those contributing to the Largest Contentful Paint (LCP) with the `priority` prop. The component adds a preload `<link>` to the document head, improving load times and enhancing your site's performance.

## Installation

Add leptos_image via cargo:

```bash
cargo add leptos_image
```

Add the SSR Feature under `[features]` in your `Cargo.toml`

```toml
[features]
ssr = [
    "leptos_image/ssr",
    # ...
 ]
```

## Quick Start

**REQUIRES SSR + AXUM** 
**(Now supports SSR + Actix-web)**

First add the provider to the base of your Leptos App.

```rust
use leptos_image::*;

#[component]
pub fn App(cx: Scope) -> impl IntoView {
    provide_image_context(cx);

    view!{
        // ...
    }
}
```

Next go to your SSR Main Function in `main.rs`

Before you create your router, call the `cache_app_images` function with the project root. This will cache all the images in your app, and generate the LQIPs.

If you have a lot of images, then you should probably only call this function in production because it will delay your server startup.

If you don't include this function, then image caching will happen at runtime.

```rust
// AXUM
use leptos::*;
use leptos_image::*;

let conf = get_configuration(None).await.unwrap();
let leptos_options = conf.leptos_options;
let root = leptos_options.site_root.clone();

use leptos_image::cache::cache_app_images;

cache_app_images(root, || view! {<App/>}, 2, || (), || ())
    .await
    .expect("Failed to cache images");

```

```rust
// ACTIX_WEB
async fn main() -> std::io::Result<()> {
    use actix_files::Files;
    use actix_web::*;
    use image_example_actix::app::*;
    use leptos::*;
    use leptos_actix::{generate_route_list, LeptosRoutes};
    use leptos_image::*;

    let conf = get_configuration(None).await.unwrap();
    let addr = conf.leptos_options.site_addr;
    // Generate the list of routes in your Leptos App
    let routes = generate_route_list(App);
    println!("listening on http://{}", &addr);
    let root = conf.leptos_options.site_root.clone();

    // run cache app images only in server

    cache_app_images(root, || view! { <App/>}, 2, || (), || ())
        .await
        .expect("Failed to cache images");

```



Next add an endpoint to your router that serves the cached images. For now, the endpoint path must be `/cache/image` and is not configurable


```rust

let router = Router::new()
        .route("/api/*fn_name", post(leptos_axum::handle_server_fns))
        .leptos_routes(&leptos_options, routes, |cx| {
            view! {
                <App/>
            }
        })
        // Here's the new route!
        .route("/cache/image", get(image_cache_handler))
        .with_state(leptos_options);

```

```rust
//ACTIX_WEB
    HttpServer::new(move || {
        let leptos_options = &conf.leptos_options;
        let site_root = &leptos_options.site_root;

        App::new()
            .app_data(web::Data::new(leptos_options.to_owned()))
            .route("/api/{tail:.*}", leptos_actix::handle_server_fns())
            .service(Files::new("/assets", site_root))
            .leptos_routes(leptos_options.to_owned(), routes.to_owned(), App)
            .service(Files::new("/pkg", format!("{site_root}/pkg")))
            // serve other assets from the `assets` directory
            .route("/cache/image", web::get().to(actix_handler))
            // serve JS/WASM/CSS from `pkg`
            // serve the favicon from /favicon.ico
            .service(favicon)

        //.wrap(middleware::Compress::default())
    })
    .bind(&addr)?
    .run()
    .await

```

Now you can use the Image Component anywhere in your app!

```rust
#[component]
pub fn MyPage(cx: Scope) -> impl IntoView {
    view! {
        <Title text="This Rust thing is pretty great"/>
        <Image
            src="/cute_ferris.png"
            blur=true
            width=750
            height=500
            quality=85
        />
    }
}
```

And that's it. You're all set to use the Image Component.

## Caveats:

- Images will only be retrieved from routes that are non-dynamic (meaning not `api/post/:id` in Route path).
- Images can take a few seconds to optimize, meaning first startup of server will be slower.



This project was folked from [leptos_image](https://github.com/nicoburniske/leptos_image)
