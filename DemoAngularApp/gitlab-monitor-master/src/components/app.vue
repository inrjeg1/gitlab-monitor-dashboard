<template>
<div class="ui-g">

  <div class="ui-g-6">
  
      <div style="background-color:SlateBlue"><h1>IOT Build Latest</h1></div>
          <div class="app" @click="autoZoom()">
            <div :style="{zoom}">
            <div class="projects">
                <project-card v-for="project in sortedProjects" :key="project.id" :project-id="project.id" v-model="project.last_activity_at" />
              </div>
            </div>
            <div v-if="initialLoading" class="loader">
              <octicon name="sync" spin scale="3" />
            </div>

          </div>
        
  </div>
    <div class="ui-g-6">
    <div style="background-color:Violet"><h1>IOT Build Stable {{stableVersion}}</h1></div>
     <div class="app" @click="autoZoom()">
            <div :style="{zoom}">
            <div class="projects">
                <project-card-stable v-for="project in sortedProjects" :key="project.id" :project-id="project.id" v-model="project.last_activity_at" />
              </div>
            </div>
            <div v-if="initialLoading" class="loader">
              <octicon name="sync" spin scale="3" />
            </div>

          </div>
    </div>

</div>

</template>

<script>
  import Octicon             from 'vue-octicon/components/Octicon';
  import {getQueryParameter} from '../util';
  import ProjectCard         from './project-card';
  import ProjectCardStable from './project-card-stable'

  export default {
    components: {
      Octicon,
      ProjectCard,
      ProjectCardStable
    },
    name: 'app',
    data: () => ({
      projects: [],
      zoom: 1,
      initialLoading: true
    }),
    computed: {
      sortedProjects() {
        //return this.$data.projects.sort((a, b) => new Date(b.last_activity_at).getTime() - new Date(a.last_activity_at).getTime());
         return this.$data.projects.sort((a, b) => a.name>b.name);

      },

      stableVersion(){
        return getQueryParameter('stableVersion')!==null ? getQueryParameter('stableVersion') :'';
      }
    },
    mounted() {
      this.fetchProjects();

      if (getQueryParameter('autoZoom')) {
        setInterval(() => {
          this.autoZoom();
        }, 5000);
      }

      this.refreshIntervalId = setInterval(async () => {
        if (!this.$data.loading) {
          await this.fetchProjects();
        }
      }, 120000);
    },
    beforeDestroy() {
      clearInterval(this.refreshIntervalId);
    },
    methods: {
      async fetchProjects() {
        const fetchCount = getQueryParameter('fetchCount') || 20;
        const gitlabApiParams = {
          order_by: 'last_activity_at',
          // GitLab per_page max is 100. We use > 100 values as next page follow trigger
          per_page: fetchCount > 100 ? 100 : fetchCount
        };

        const visibility = getQueryParameter('projectVisibility') || 'any';
        // Only add the visibility attribute to the params if filtering is required
        // (if visiblity is not specified, Gitlab will return all projects)
        if (visibility !== 'any') {
          gitlabApiParams.visibility = visibility;
        }

        const projects = await this.$api('/projects', gitlabApiParams, {follow_next_page_links: fetchCount > 100});

        // Only show projects that have jobs enabled
        const maxAge = (getQueryParameter('maxAge') !== null ? getQueryParameter('maxAge') : 24 * 7);
        //const maxAge = 3000;

        // Only include projects from specific groups
        let includeGroups = (getQueryParameter('groups') !== null ? getQueryParameter('groups') : '');
        if (typeof includeGroups === 'string') {
          includeGroups = includeGroups.split(',');
        }

        // Only include specific projects
        let includeProjects = (getQueryParameter('projects') !== null ? getQueryParameter('projects') : '');
        if (typeof includeProjects === 'string') {
          includeProjects = includeProjects.split(',');
        }

        this.$data.projects = projects.filter((project) => {
          return project.jobs_enabled &&
            (maxAge === 0 || ((new Date() - new Date(project.last_activity_at)) / 1000 / 60 / 60 <= maxAge)) &&
            (
              (includeGroups[0] === '' && includeProjects[0] === '') ||
              (includeGroups.some((group) => group === project.namespace.name)) ||
              (includeProjects.some((group) => group === project.name_with_namespace))||
              (includeProjects.some((group) => group === project.name))
            );
        });

        if (getQueryParameter('autoZoom')) {
          this.$nextTick(() => this.autoZoom());
        }

        this.$data.initialLoading = false;
      },
      async autoZoom() {
        let step = 0.1;

        if (this.$el.clientHeight > window.innerHeight) {
          step = -0.1;
        }

        while (
          (step > 0 && this.$el.clientHeight <= window.innerHeight) ||
          (step < 0 && this.$el.clientHeight > window.innerHeight)
        ) {
          this.$data.zoom += step;
          await this.$nextTick();

          if (this.$data.zoom > 20 || this.$data.zoom < 0) {
            // The browser likely doesn't support CSS zoom
            this.$data.zoom = 0;
            return;
          }
        }

        if (step > 0) this.$data.zoom -= step;
      }
    }
  };
</script>

<style lang="scss">
@import '../../node_modules/primeng/resources/primeng.min.css';
  html {
    background-color: #212121;
    color: #dddddd;
    font-family: Roboto, sans-serif;
  }

  body {
    margin: 0;
    padding: 4px;
    overflow-y: hidden;
  }

  svg {
    overflow: visible;
  }

  .fade-enter-active, .fade-leave-active {
    transition: opacity 0.15s
  }

  .fade-enter, .fade-leave-to {
    opacity: 0
  }
  .header{
    overflow: hidden;
    background-color:#f1f1f1
  }
  h3{
    color:blue;
  }
</style>

<style lang="scss" scoped>
  .app {
    .projects {
      display: flex;
      flex-wrap: wrap;
      justify-content: left;
    }

    .loader {
      display: flex;
      justify-content: center;
      align-items: center;
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background-color: transparentize(#212121, 0.5);
    }
  }
</style>
