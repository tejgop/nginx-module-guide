---
title: Tejas's Nginx Module Guide

search: true
---

# Introduction

Welcome to my Nginx module guide!


To follow this guide, you need to know a decent amount of C. You should know
about structs, pointers, and functions. You also need to know how the
nginx.conf file works.


If you find a mistake in the guide, please report it in an [issue](https://github.com/tejgop/nginx-module-guide/issues)!

<aside class="warning">
This guide is currently being written. Use it at your own risk!
</aside>


# The Handler Guide

Let's get started with a quick hello world module called
`ngx_http_hello_world_module`.

This module will be a handler, meaning that it will take a request and
generate output.

In the nginx source, create a folder called `ngx_http_hello_world_module`,
and make two files in it: `config` and `ngx_http_hello_world_module.c`.

## The Config File

> config

```shell
ngx_addon_name=ngx_http_hello_world_module

if test -n "$ngx_module_link"; then
    # The New Way
    ngx_module_type=HTTP
    ngx_module_name=ngx_http_hello_world_module
    ngx_module_srcs="$ngx_addon_dir/ngx_http_hello_world_module.c"

    . auto/module
else
    # The Old Way
    HTTP_MODULES="$HTTP_MODULES ngx_http_hello_world_module"
    NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_hello_world_module.c"
fi
```

The config file is just a simple shell script that will be used at compile
time to show Nginx where your module source is. As you can see, the config
file tests to see if your nginx version supports dynamic modules
(the `test -n` line). If it supports dynamic modules, the module is added
the new way. Otherwise, it is added the old way.


## The C File

> ngx_http_hello_world_module.c

```c

#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

static char *ngx_http_hello_world(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);

static ngx_command_t  ngx_http_hello_world_commands[] = {
  {
    ngx_string("print_hello_world"),
    NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
    ngx_http_hello_world,
    0,
    0,
    NULL
  },
    ngx_null_command
};

static ngx_http_module_t  ngx_http_hello_world_module_ctx = {
  NULL,
  NULL,
  NULL,
  NULL,
  NULL,
  NULL,
  NULL,
  NULL
};

ngx_module_t ngx_http_hello_world_module = {
  NGX_MODULE_V1,
  &ngx_http_hello_world_module_ctx,
  ngx_http_hello_world_commands,
  NGX_HTTP_MODULE,
  NULL,
  NULL,
  NULL,
  NULL,
  NULL,
  NULL,
  NULL,
  NGX_MODULE_V1_PADDING
};

static ngx_int_t ngx_http_hello_world_handler(ngx_http_request_t *r)
{
  u_char *ngx_hello_world = (u_char *) "Hello World!";
  size_t sz = sizeof(ngx_hello_world);

  r->headers_out.content_type.len = sizeof("text/html") - 1;
  r->headers_out.content_type.data = (u_char *) "text/html";
  r->headers_out.status = NGX_HTTP_OK;
  r->headers_out.content_length_n = sz;
  ngx_http_send_header(r);

  ngx_buf_t    *b;
  ngx_chain_t   *out;

  b = ngx_calloc_buf(r->pool);

  out = ngx_alloc_chain_link(r->pool);

  out->buf = b;
  out->next = NULL;

  b->pos = ngx_hello_world;
  b->last = ngx_hello_world + sz;
  b->memory = 1;
  b->last_buf = 1;

  return ngx_http_output_filter(r, out);
}

static char *ngx_http_hello_world(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
  ngx_http_core_loc_conf_t  *clcf;
  clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
  clcf->handler = ngx_http_hello_world_handler;
  return NGX_CONF_OK;
}

```

This C file is huge! Let's go through it line by line:


The first line is a prototype for the function `ngx_http_hello_world`.
We'll define the function at the end of the file.


`ngx_http_hello_world_commands` is a static array of directives. In our
module, we have only one directive: `print_hello_world`. It will have no
arguments, so we put in `NGX_CONF_NOARGS`.


`ngx_http_hello_world_module_ctx` is an array of function references.
The functions will be executed for various purposes such as preconfiguration,
postconfiguration, etc. We don't need this array in our module, but we
still have to define it and fill it with `NULL`s.


`ngx_http_hello_world_module` is an array of definitions for the
module. It tells where the array of directives and functions are
(`ngx_http_hello_world_module` and `ngx_http_hello_world_module_ctx`).
We can also add init and exit callback functions. In our module, we don't need them so
we put `NULL`s instead.


Now for the interesting part. `ngx_http_hello_world_handler` is the heart
of our module. We want to print `Hello World!` on the screen, so we have
an `unsigned char *` with our message in it. Right after that, there
is another variable with the size of the message.


Next, we have to send the headers. Notice that `ngx_http_hello_world_handler`
had 1 argument that was of type `ngx_http_request_t`. This is a custom
struct made by Nginx. It has a member called `headers_out`, which
we use to send the headers. After we are done setting the headers, we
can send them with `ngx_http_send_header(r)`.


Now we have to send the body. `ngx_buf_t` is a buffer, and `ngx_chain_t`
is a chain link. The chain links send responses buffer by buffer and point
to the next link. In our module, there is no next link, so we set `out->next`
to `NULL`. `ngx_calloc_buf` and `ngx_alloc_chain_link` are Nginx's
calloc wrappers that automatically take
care of garbage collection. `b->pos` and `b->last` help us send our
content. `b->pos` is the first position in the memory and `b->last` is
the last position. `b->memory` is set to 1 because our content
is read-only. `b->last_buf` tells that our buffer is the last buffer
in the request.


Now that we're done setting the body, we can send it with
`return ngx_http_output_filter(r, &out)`


Now we define that function we prototyped in the beginning.
We can show Nginx what our handler is called with
`clcf-> handler = ngx_http_hello_world_handler`.


And we're done with our C file! Time to build the module.


## Building the Module

> How to build the module:

```bash
$ ./configure \
> --prefix=/where/i/want/to/install/nginx \
> --add-dynamic-module=/path/to/ngx_http_hello_world_module

# Build module and Nginx

$ make
$ make install

# Build module

$ make modules
$ cp objs/ngx_http_hello_world_module.so <nginx_install_location>/modules
```

In the Nginx source, run `configure`, `make`, and `make install`. If you only
want to build the modules and not the Nginx server itself, you can run
`make modules`.


<aside class="notice">
If you run make modules, you have to manually move your module
*.so file from the objs directory to your Nginx installation.
</aside>

## Using the Module

> nginx.conf

```
load_module "modules/ngx_http_hello_world_module.so"

http {
  default_type  application/octet-stream;
  server {
    listen 8000;
    server_name localhost;
    location / {
      root  html;
      index index.html index.htm;
    }
    location /test {
      print_hello_world;
    }
  }
}
```

To use the module, edit your `nginx.conf` file found in the `conf` directory
in the install location.


When you're done, you can run nginx (`<nginx_install_location>/sbin/nginx`)
and take a look at your work at
[localhost:8000/test](http://localhost:8000/test). You should get a blank
page saying `Hello World!`. If so, congratulations! You made your first
Nginx module! This module is the base for making any handler.


## Printing All the URL Arguments

> Modified ngx_http_hello_world_handler:

```c
static ngx_int_t ngx_http_hello_world_handler(ngx_http_request_t *r)
{
  u_char *ngx_hello_world = r->args.data;
  size_t sz = r->args.len;

  r->headers_out.content_type.len = sizeof("text/html") - 1;
  r->headers_out.content_type.data = (u_char *) "text/html";
  r->headers_out.status = NGX_HTTP_OK;
  r->headers_out.content_length_n = sz;
  ngx_http_send_header(r);

  ngx_buf_t    *b;
  ngx_chain_t   *out;

  b = ngx_calloc_buf(r->pool);

  out = ngx_alloc_chain_link(r->pool);

  out.buf = b;
  out.next = NULL;

  b->pos = ngx_hello_world;
  b->last = ngx_hello_world + sz;
  b->memory = 1;
  b->last_buf = 1;

  return ngx_http_output_filter(r, &out);
}
```

Now, we'll modify our module slightly to print all the URL arguments
(everything after the `?`). So if our request is
`localhost:8000/test?foo=bar&hello=world` we should get `foo=bar&hello=world`
printed in the body.


We need to modify the handler, `ngx_http_hello_world_handler`. Notice
that the string `Hello World!` was changed to `r->args.data`, and
`sizeof(ngx_hello_world)` was changed to `r->args.len`. `r->args`stores
all the arguments and is of type `ngx_str_t`. `ngx_str_t`s have a `data`
and a `len` element, for storing the string and its length.


When you're done, you should stop nginx
(`<nginx_install_location>/sbin/nginx> -s stop`) and build again. After that's
done, start Nginx again and go to
[localhost:8000/test?foo=hello&bar=world](localhost:8000/test?foo=hello&bar=world).
You should see `foo=hello&bar=world` printed in the body.

## Many Buffers

> Modified ngx_http_hello_world_handler:

```c
static ngx_int_t ngx_http_hello_world_handler(ngx_http_request_t *r)
{
  u_char *ngx_hello_world = (u_char *) "Hello World!";
  size_t sz = sizeof(ngx_hello_world);

  r->headers_out.content_type.len = sizeof("text/html") - 1;
  r->headers_out.content_type.data = (u_char *) "text/html";
  r->headers_out.status = NGX_HTTP_OK;
  r->headers_out.content_length_n = sz;
  ngx_http_send_header(r);

  ngx_buf_t    *b, *b2;
  ngx_chain_t   *out, *out2;

  b = ngx_calloc_buf(r->pool);
  b2 = ngx_calloc_buf(r->pool);

  out = ngx_alloc_chain_link(r->pool);
  out2 = ngx_alloc_chain_link(r->pool);

  b->pos = ngx_hello_world;
  b->last = ngx_hello_world + sz;
  b->memory = 1;
  b->last_buf = 0;

  b2->pos = ngx_hello_world;
  b2->last = ngx_hello_world + sz;
  b2->memory = 1;
  b2->last_buf = 1;

  out = 

  out2->buf = b2;
  out2->next = NULL;

  out->buf = b;
  out->next = out2;


  return ngx_http_output_filter(r, out);
}

```


Get your hello world template again and add another buffer
(`ngx_buf_t *b2`) and another chain link (`ngx_chain_t out2;`).
Then allocate some memory with `ngx_calloc_buf` and `ngx_alloc_chain_link`.
Everything is the same except
that we are setting `b->last_buf` to 0 and `out.next` to `out2`. This
is because our original buffer is no longer the last one, and the next
buffer is `out2`.

Now, if we send the headers we only mention our original buffer because
`out` links to the next buffer.

Rebuild, restart, and go to `localhost:8000/test`. You should see
`Hello World!Hello World!`.


# The Filter Guide

After a handler is loaded and run, all the filter modules are executed.
Filters take the header and/or body, manipulate them, and then send them back. 

Our module will add a music track to all web pages where the module is loaded.

## The Config File

> config

```shell
ngx_addon_name=ngx_http_hello_world_module

if test -n "$ngx_module_link"; then
    # The New Way
    ngx_module_type=HTTP_FILTER
    ngx_module_name=ngx_http_hello_world_module
    ngx_module_srcs="$ngx_addon_dir/ngx_http_hello_world_module.c"

    . auto/module
else
    # The Old Way
    HTTP_FILTER_MODULES="$HTTP_FILTER_MODULES ngx_http_hello_world_module"
    NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_hello_world_module.c"
fi
```

Nothing is very different here, except that `HTTP` is replaced with `HTTP_FILTER`.

## The C File

> ngx_http_hello_world_module.c

```c
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

u_char *hello_world = (u_char *) "<audio controls loop autoplay src=\"https://upload.wikimedia.org/wikipedia/commons/8/85/Holst-_mars.ogg\"></audio>";

static char *
ngx_http_hello_world(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
  return NGX_OK;
}
static ngx_int_t ngx_http_hello_world_init(ngx_conf_t *cf);


static ngx_command_t  ngx_http_hello_world_commands[] = {
  
  { ngx_string("hello_world"),
  NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
  ngx_http_hello_world,
  0,
  0,
  NULL },
  
  ngx_null_command
};


static ngx_http_module_t  ngx_http_hello_world_module_ctx = {
  NULL,                                         /* proconfiguration */
  ngx_http_hello_world_init,               /* postconfiguration */
  
  NULL,                                         /* create main configuration */
  NULL,                                         /* init main configuration */
  
  NULL,                                         /* create server configuration */
  NULL,                                         /* merge server configuration */
  
  NULL,					        /* create location configuration */
  NULL					        /* merge location configuration */
};




ngx_module_t  ngx_http_hello_world_module = {
  NGX_MODULE_V1,
  &ngx_http_hello_world_module_ctx, /* module context */
  ngx_http_hello_world_commands,    /* module directives */
  NGX_HTTP_MODULE,                       /* module type */
  NULL,                                  /* init master */
  NULL,                                  /* init module */
  NULL,                                  /* init process */
  NULL,                                  /* init thread */
  NULL,                                  /* exit thread */
  NULL,                                  /* exit process */
  NULL,                                  /* exit master */
  NGX_MODULE_V1_PADDING
};




static ngx_http_output_header_filter_pt ngx_http_next_header_filter;
static ngx_http_output_body_filter_pt   ngx_http_next_body_filter;


static ngx_int_t
ngx_http_hello_world_header_filter(ngx_http_request_t *r)
{
  
  r->headers_out.content_length_n += strlen((char *)hello_world);
  
  return ngx_http_next_header_filter(r);
}


static ngx_int_t
ngx_http_hello_world_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
  
  ngx_buf_t             *buf;
  ngx_chain_t           *link;
  
  
  buf = ngx_calloc_buf(r->pool);
  
  buf->pos = hello_world;
  buf->last = buf->pos + strlen((char *)hello_world);
  buf->start = buf->pos;
  buf->end = buf->last;
  buf->last_buf = 0;
  buf->memory = 1;
  
  link = ngx_alloc_chain_link(r->pool);
  
  link->buf = buf;
  link->next = in;
  
  return ngx_http_next_body_filter(r, link);
}

static ngx_int_t
ngx_http_hello_world_init(ngx_conf_t *cf)
{
  ngx_http_next_body_filter = ngx_http_top_body_filter;
  ngx_http_top_body_filter = ngx_http_hello_world_body_filter;
  
  ngx_http_next_header_filter = ngx_http_top_header_filter;
  ngx_http_top_header_filter = ngx_http_hello_world_header_filter;
  
  return NGX_OK;
}

```

Let's see how this is different from our handler. First we see our
`u_char`: An HTML `<audio>` element with a song (from Wikipedia).


Next, instead of prototyping `ngx_http_background_music`, we
just defined it. Normally, this function would be more interesting,
but since this is just a demo module, we don't need to do anything
other than returning `NGX_OK`.


The filter is made of two parts: the header filter, and the body filter.
For our header filter, we only have to add the content length
of our `background_music` variable. After that, we pass on the baton
to the next header filter with `ngx_http_next_header_filter`.


The body filter accepts 2 arguments: An `ngx_http_request_t`, and a
chain link, `ngx_chain_t`. The chain link is from the handler and
previous filters (if any). What we want to do is to prefix our
audio element to the chain link `in`. It won't be perfectly valid
HTML, but it's good enough for now.


Our buffer should have `last_buf` set to `0` because it isn't the last
buffer: The last buffer is in the `in` chain link. So we'll just set
`link->next` to `in` and call the next body filter.


The `ngx_http_background_music_init` just tells what our filter
funtions are called.


And we're done. Now build and reload nginx, and you should see...
a 404 page? Yes, there will be a 404 page, but with an audio track
above it.


## Using the Module

> nginx.conf

```
load_module "modules/ngx_http_hello_world_module.so"

http {
  default_type  application/octet-stream;
  server {
    listen 8000;
    server_name localhost;
    location / {
      root  html;
      index index.html index.htm;
    }
    location /test {
      root html;
      index index.html index.htm;
      print_hello_world;
    }
  }
}
```

There was a 404 page because there was no other handler given.
If you just add an HTML file, that should be taken care of.


# Data Types

## ngx_http_request_t

> Example usage of the server variable:

```c
function hello_world_handler(ngx_http_request_t *r){
  ...
  ngx_str_t *test = r->server;
  u_char *testString = test.data;
  ...
}
```

This table lists some useful members of `ngx_http_request_t`

Member|Type|Description
--------|----|----------
args|ngx_str_t|All the arguments in one string
server|ngx_str_t|The server name
uri|ngx_str_t|The request path
pool|ngx_pool_t*|Used for allocating memory

## ngx_str_t

> ngx_str_t usage:

```c
// Initialize string
ngx_str_t mystring = ngx_string("hello");

// Use string
...
r->pos = mystring.data;
r->last = mystring.data + mystring.len;
...
```

The `ngx_str_t` datatype has 2 members: `data` and `len`. They allow
you to access the contents of the string and it's length.

Member|Type|Description
------|----|-----------
data|u_char*|The contents of the string
len|size_t|The length of the string

# Other Stuff

## Troubleshooting

### 1. Only part of my text is showing up!

You have to correctly set the size in both the headers and
in the buffer. If your string is a `u_char*`, use `ngx_strlen`, if it's
an `ngx_str_t`, use `{variable_name}.len`.

### 2. Some weird string is showing up after my text.

See #1.

### 3. Why is my filter not working (but compiling)?

Make sure you've configured it correctly. The `config` file
should have `HTTP_FILTER` instead of `HTTP`. Also, check if
you've sent the body correctly, and make sure your buffer is in
the chain link.

## Useful Links

- [Emiller's Guide](http://www.evanmiller.org/nginx-modules-guide.html)
- [Nginx Main Module API](https://www.nginx.com/resources/wiki/extending/api/main/)
- [Nginx Memory Management API](https://www.nginx.com/resources/wiki/extending/api/alloc/)
