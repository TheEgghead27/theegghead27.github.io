<!-- 
	Blog index gets its own page design
	
	<ContentDoc> uses the first content it finds,
	so we can't use `index.md` reliably.
-->
<template>
  <nav>
    <ContentNavigation v-slot="{ navigation }">
      <div v-for="blog of navigation" :key="blog._path">
        <div v-for="post of blog.children" :key="post._path">
          <NuxtLink :to="post._path">{{ post.title }}</NuxtLink>
      	</div>
      </div>
    </ContentNavigation>
  </nav>
  <main>
    Welcome to the blog!
    A properly designed catalog page will appear here eventually...
  </main>
</template>
