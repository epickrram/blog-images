In my last post, I looked at annotating Flamegraphs with contextual information in order to
aid in filtering on an interesting subset of the data. One of the things that stuck with me
was the idea of using SVGs to render data generated on a server that runs in headless mode.

Traditionally, I have recorded profiles and traces on remote servers, then pulled the data
back to my workstation to filter, aggregate and plot. The scripts used to do this data munging
tend to be one-shot affairs, and I've probably lost many useful utilities over the years. I
really like the idea of building the rendering into the server-side script, as it forces us to 
think about how we want to interpret the data, and also gives us the ability to deploy and serve
such monitoring from a whole fleet of servers.

Partly to address this, and partly because experimentation is fun, I've been working on some
new visualisations in the same vein as Flamegraphs. 



