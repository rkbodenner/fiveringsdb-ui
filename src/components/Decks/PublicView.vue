<template>
    <div class="deck">
        <div v-if="loading" class="loading">
            Loading...
        </div>

        <div v-if="error" class="alert alert-danger" role="alert">
            {{ error }}
        </div>

        <div v-if="deck" class="content" :class="['link-'+deck.primary_clan]">
            <div class="text-light p-2 mb-4" :class="['bg-dark-'+deck.primary_clan]">
                <h1 class="text-center pt-4">{{ deck.name }}</h1>
                <div class="small d-flex justify-content-between align-items-end">
                    <span>
                        {{ $t('format.' + deck.format) }}
                    </span>
                    <span>
                        published {{ fromNow(deck.updated_at) }}
                    </span>
                </div>
            </div>

            <div class="row">
                <div class="col-lg-6">
                    <utils-deck-content :deck="deck" :editable="false"></utils-deck-content>

                    <div class="pb-1 my-3 bb-10 bt-10 d-flex justify-content-around">
                        <a v-if="liked === false"
                           title="Like"
                           href="#"
                           @click.prevent="like"
                           role="button"
                           class="btn btn-link text-danger">
                            <span class="fa fa-heart-o"></span>
                            Like
                        </a>
                        <a v-if="liked === true"
                           title="Cancel Like"
                           href="#"
                           @click.prevent="unlike"
                           role="button"
                           class="btn btn-link text-danger">
                            <span class="fa fa-heart"></span>
                            Liked
                        </a>
                        <a href="#new-comment" class="btn btn-link text-success">
                            <span class="fa fa-comment-o"></span>
                            Comment
                        </a>

                    </div>
                </div>
                <div class="col-lg-6">
                    <h3>
                        <span class="fa fa-user-circle-o"></span> {{ deck.user.username }}
                    </h3>

                    <div class="desc-description" v-html="description"></div>

                    <div v-if="author" class="pb-1 mt-3 bt-10 d-flex justify-content-around">
                        <a href="#"
                           @click.prevent="patch"
                           role="button"
                           title="Edit the description or name of this deck"
                           class="btn btn-link text-primary">
                            <span class="fa fa-pencil"></span>
                            Edit
                        </a>
                        <button
                           v-b-modal.modalDelete
                           role="button"
                           :disabled="!removable"
                           :title="removable ? 'Delete this deck' : 'This deck cannot be deleted because it has comments'"
                           class="btn btn-link text-danger">
                            <span class="fa fa-trash"></span>
                            Delete
                        </button>
                    </div>

                    <b-modal id="modalDelete"
                             ref="modalDelete"
                             title="Confirmation"
                             @ok="remove"
                             ok-title="Confirm deletion"
                             close-title="Cancel">
                        <p>This deletion is definitive and cannot be undone.</p>
                        <p>Delete <b>{{ deck.name }}</b>?</p>
                    </b-modal>

                    <comments-list :deck="deck"></comments-list>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
  import moment from 'moment';
  import MarkdownIt from 'markdown-it';
  import rest from '@/rest';
  import UtilsDeckContent from '@/components/Utils/DeckContent';
  import CommentsList from '../Comments/List';

  const md = new MarkdownIt();

  export default {
    name: 'public-decks-view',
    components: {
      UtilsDeckContent,
      CommentsList,
    },
    data() {
      return {
        loading: false,
        deck: null,
        error: null,
        comment: '',
        liked: false,
      };
    },
    watch: {
      $route: 'fetchData',
    },
    computed: {
      removable() {
        return this.deck.comments.length === 0;
      },
      description() {
        return md.render(this.deck.description);
      },
      content() {
        return this.$store.getters.slots;
      },
      author() {
        return this.deck && this.deck.user.id === this.$store.getters.userId;
      },
    },
    methods: {
      like() {
        if (this.$store.getters.isLogged) {
          rest
            .post(`decks/${this.$route.params.deckId}/likes`)
            .then(() => {
              this.$notify({
                title: 'Success',
                text: 'Liked!',
                type: 'success',
              });
              this.liked = true;
            })
            .catch((reason) => {
              this.$notify({
                title: 'Error',
                text: reason,
                type: 'error',
              });
            });
        }
      },
      unlike() {
        if (this.$store.getters.isLogged) {
          rest
            .delete(`decks/${this.$route.params.deckId}/likes`)
            .then(() => {
              this.$notify({
                title: 'Success',
                text: 'Cancelled!',
                type: 'success',
              });
              this.liked = false;
            })
            .catch((reason) => {
              this.$notify({
                title: 'Error',
                text: reason,
                type: 'error',
              });
            });
        }
      },
      fromNow(date) {
        return moment(date).fromNow();
      },
      fetchData() {
        this.error = null;
        this.deck = null;
        this.loading = true;
        rest
          .get(`decks/${this.$route.params.deckId}`)
          .then((result) => {
            this.loading = false;
            this.deck = result.record;
          })
          .catch((reason) => {
            this.loading = false;
            this.error = reason;
          });
        if (this.$store.getters.isLogged) {
          rest
            .get(`decks/${this.$route.params.deckId}/likes/me`)
            .then((result) => {
              this.liked = result.hasOwnProperty('record');
            })
            .catch((reason) => {
              this.$notify({
                title: 'Error',
                text: reason,
                type: 'error',
              });
            });
        }
      },
      patch() {
        this.$router.push({ name: 'deck-patch', params: { deckId: this.deck.id } });
      },
      remove() {
        rest
          .delete(`decks/${this.$route.params.deckId}`)
          .then(() => {
            this.$router.push({ name: 'deckbuilder' });
            this.$notify({
              title: 'Success',
              text: 'Deleted!',
              type: 'success',
            });
          })
          .catch((reason) => {
            this.$notify({
              title: 'Error',
              text: reason,
              type: 'error',
            });
          });
      },
    },
    created() {
      this.fetchData();
    },
    mounted() {
      this.$store.commit('changeDocumentTitle', 'Deck Browser');
    },
  };
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>
</style>
