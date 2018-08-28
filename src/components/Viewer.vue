<template>
    <div id="viewer">
        <div id="aladin-lite-div"></div>
    </div>
</template>

<script lang="ts">
import { Vue, Component, Prop } from 'vue-property-decorator';

// A is a global variable from aladin lite source code.
// It is declared for the typescript checker to understand it.
declare var A: any;

// Defining interface objects for the different events
// linked to Aladin-Lite.
interface AddHiPSCatalogData {
  ID: string;
  hips_service_url: string;
}

interface AddHiPSSurveyData {
  ID: string;
  hips_service_url: string;
  hips_order: string;
  hips_tile_format: string;
}

interface AddMOCData {
  moc_access_url: string;
}

interface AddVizierCatalogData {
  ID: string;
  obs_id: string;
}

@Component({
    name: 'ViewerComponent',
})
export default class ViewerComponent extends Vue {
  /* The reference to the aladin-lite view is private.
      The view must only be changed by the methods of the ViewerComponent. */
  private aladin: any;

  private loadScriptIntoDOM(bodyElement: HTMLElement, url: string, onloadCallback?: () => void) {
    const scriptElement = document.createElement('script');
    scriptElement.setAttribute('src', url);
    scriptElement.async = false;
    if (onloadCallback) {
      scriptElement.onload = onloadCallback;
    }

    bodyElement.appendChild(scriptElement);
  }

  private mounted() {
    // Now the component is mounted we can load aladin lite.
    const bodyElement = document.getElementsByTagName('BODY')[0] as HTMLElement;
    // jQuery is a dependency for aladin-lite and therefore must be inserted in the DOM.
    this.loadScriptIntoDOM(bodyElement, 'http://code.jquery.com/jquery-1.12.1.min.js');
    // Then we load the aladin lite script.
    this.loadScriptIntoDOM(bodyElement, 'http://aladin.u-strasbg.fr/AladinLite/api/v2/beta/aladin.min.js', () => {
      // When the import has succeded we store the aladin js instance into its component
      this.aladin = A.aladin('#aladin-lite-div', {survey: 'P/DSS2/color', fov: 60});

      /* Event definitions. When one of the events defined below are emitted from other components, this component
          will catch them and execute a bunch of methods from the API of the viewer. This practise encapsulates the
          API of the viewer and then aims to be extended for other viewer than Aladin-lite (e.g. firefly). */

      // Event for adding an HiPS catalog (e.g. gaia2) in Aladin-Lite.
      this.$root.$on('event-add-hips-catalogs', (metadata: AddHiPSCatalogData) => {
        const hips = A.catalogHiPS(metadata.hips_service_url, {
          onClick: 'showTable',
          name: metadata.ID,
        });
        this.aladin.addCatalog(hips);
      });

      // Event for adding an HiPS image (e.g. DSS2/color) in Aladin-Lite.
      this.$root.$on('event-add-hips-images', (metadata: AddHiPSSurveyData) => {
        const order = (+metadata.hips_order);
        const hipsTileFormat = metadata.hips_tile_format.split(' ');
        let tileFormat = hipsTileFormat[0];
        if (tileFormat === 'jpeg') {
          tileFormat = 'jpg';
        }
        const imageSurvey = this.aladin.createImageSurvey(metadata.ID,
          metadata.ID,
          metadata.hips_service_url,
          'j2000',
          order, {
            imgFormat: tileFormat,
          },
        );
        this.aladin.setImageSurvey(imageSurvey);
      });

      // Event for adding a multi-order coverage map of a survey/catalog in Aladin-Lite.
      this.$root.$on('event-add-hips-moc', (metadata: AddMOCData) => {
        const moc = A.MOCFromURL(metadata.moc_access_url, {
          color: '#84f',
          lineWidth: 1,
        });
        this.aladin.addMOC(moc);
      });

      // Event for adding a vizier catalog in Aladin-Lite.
      this.$root.$on('event-add-catalogs-from-vizier', (metadata: AddVizierCatalogData) => {
        const centerPos = this.aladin.getRaDec();
        const cornerPos = this.aladin.getFovCorners()[0];
        const dist = Math.sqrt(Math.pow(centerPos[0] - cornerPos[0], 2) + Math.pow(centerPos[1] - cornerPos[1], 2));
        const catalog = A.catalogFromVizieR(metadata.obs_id, this.aladin.getRaDec(), dist, {
          onClick: 'showTable',
          limit: 1000,
        });

        this.aladin.addCatalog(catalog);
      });

      const sendFovCorners = () => {
        this.$root.$emit('event-update-fov-corners', this.aladin.getFovCorners());
      };

      // If the user zoom or change the view we emit an event giving the new positions of the FOV corners
      // to the other components.
      this.aladin.on('zoomChanged', sendFovCorners);
      this.aladin.on('positionChanged', sendFovCorners);
    });
  }
}
</script>
<style>
/* Load the CSS file of aladin-lite */
@import "http://aladin.u-strasbg.fr/AladinLite/api/v2/latest/aladin.min.css";

</style>
