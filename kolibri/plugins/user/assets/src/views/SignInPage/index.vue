<template>

  <div class="fh">

    <FacilityModal
      v-if="facilityModalVisible"
      @close="closeFacilityModal"
    />

    <div class="wrapper-table">
      <div class="main-row">
        <div class="main-cell">
          <div class="box">
            <CoreLogo
              class="logo"
              :style="{'height': `${logoHeight}px`}"
            />
            <h1 :style="{'font-size': `${logoTextSize}px`}">
              {{ $tr('kolibri') }}
            </h1>
            <form class="login-form" ref="form" @submit.prevent="signIn">
              <UiAlert
                v-if="invalidCredentials"
                type="error"
                :dismissible="false"
              >
                {{ $tr('signInError') }}
              </UiAlert>
              <transition name="textbox">
                <KTextbox
                  ref="username"
                  id="username"
                  autocomplete="username"
                  :autofocus="!hasMultipleFacilities"
                  :label="$tr('username')"
                  :invalid="usernameIsInvalid"
                  :invalidText="usernameIsInvalidText"
                  @blur="handleUsernameBlur"
                  @input="showDropdown = true"
                  @keydown="handleKeyboardNav"
                  v-model="username"
                />
              </transition>
              <transition name="list">
                <ul
                  v-if="simpleSignIn && suggestions.length"
                  v-show="showDropdown"
                  class="suggestions"
                >
                  <UiAutocompleteSuggestion
                    v-for="(suggestion, i) in suggestions"
                    :key="i"
                    :suggestion="suggestion"
                    :class="{ highlighted: highlightedIndex === i }"
                    @click.native="fillUsername(suggestion)"
                  />
                </ul>
              </transition>
              <transition name="textbox">
                <KTextbox
                  v-if="needPasswordField"
                  ref="password"
                  id="password"
                  type="password"
                  autocomplete="current-password"
                  :label="$tr('password')"
                  :autofocus="simpleSignIn"
                  :invalid="passwordIsInvalid"
                  :invalidText="passwordIsInvalidText"
                  :floatingLabel="!autoFilledByChromeAndNotEdited"
                  @blur="passwordBlurred = true"
                  @input="handlePasswordChanged"
                  v-model="password"
                />
              </transition>
              <div>
                <KButton
                  class="login-btn"
                  type="submit"
                  :text="$tr('signIn')"
                  :primary="true"
                  :disabled="busy"
                />
              </div>
            </form>

            <KRouterLink
              v-if="canSignUp"
              class="create-button"
              :text="$tr('createAccount')"
              :to="signUpPage"
              :primary="true"
              appearance="flat-button"
            />
            <div>
              <KExternalLink
                v-if="allowGuestAccess"
                class="guest-button"
                :text="$tr('accessAsGuest')"
                :href="guestURL"
                :primary="true"
                appearance="basic-link"
              />
            </div>
            <p class="version">{{ versionMsg }}</p>
          </div>
        </div>
      </div>
      <div class="footer-row">
        <LanguageSwitcherFooter class="footer-cell" />
      </div>
    </div>

  </div>

</template>


<script>

  import { mapState, mapGetters, mapActions } from 'vuex';
  import { FacilityUsernameResource } from 'kolibri.resources';
  import { LoginErrors } from 'kolibri.coreVue.vuex.constants';
  import KButton from 'kolibri.coreVue.components.KButton';
  import KRouterLink from 'kolibri.coreVue.components.KRouterLink';
  import KExternalLink from 'kolibri.coreVue.components.KExternalLink';
  import KTextbox from 'kolibri.coreVue.components.KTextbox';
  import CoreLogo from 'kolibri.coreVue.components.CoreLogo';
  import UiAutocompleteSuggestion from 'keen-ui/src/UiAutocompleteSuggestion';
  import UiAlert from 'keen-ui/src/UiAlert';
  import responsiveWindow from 'kolibri.coreVue.mixins.responsiveWindow';
  import urls from 'kolibri.urls';
  import { PageNames } from '../../constants';
  import LanguageSwitcherFooter from '../LanguageSwitcherFooter';
  import FacilityModal from './FacilityModal';

  export default {
    name: 'SignInPage',
    $trs: {
      kolibri: 'Kolibri',
      signIn: 'Sign in',
      username: 'Username',
      password: 'Password',
      enterPassword: 'Enter password',
      createAccount: 'Create an account',
      accessAsGuest: 'Continue as guest',
      signInError: 'Incorrect username or password',
      poweredBy: 'Kolibri {version}',
      required: 'This field is required',
      requiredForCoachesAdmins: 'Password is required for coaches and admins',
      documentTitle: 'User Sign In',
    },
    metaInfo() {
      return {
        title: this.$tr('documentTitle'),
      };
    },
    components: {
      KButton,
      KRouterLink,
      KExternalLink,
      KTextbox,
      FacilityModal,
      CoreLogo,
      UiAutocompleteSuggestion,
      UiAlert,
      LanguageSwitcherFooter,
    },
    mixins: [responsiveWindow],
    data() {
      return {
        username: '',
        password: '',
        usernameSuggestions: [],
        facilityModalVisible: this.$store.state.signIn.hasMultipleFacilities,
        suggestionTerm: '',
        showDropdown: true,
        highlightedIndex: -1,
        usernameBlurred: false,
        passwordBlurred: false,
        formSubmitted: false,
        autoFilledByChromeAndNotEdited: false,
      };
    },
    computed: {
      ...mapGetters(['facilityConfig']),
      // backend's default facility on load
      ...mapState(['facilityId']),
      ...mapState('signIn', ['hasMultipleFacilities']),
      ...mapState({
        passwordMissing: state => state.core.loginError === LoginErrors.PASSWORD_MISSING,
        invalidCredentials: state => state.core.loginError === LoginErrors.INVALID_CREDENTIALS,
        busy: state => state.core.signInBusy,
      }),
      simpleSignIn() {
        return this.facilityConfig.learnerCanLoginWithNoPassword;
      },
      suggestions() {
        // Filter suggestions on the client side so we don't hammer the server
        return this.usernameSuggestions.filter(sug =>
          sug.toLowerCase().startsWith(this.username.toLowerCase())
        );
      },
      usernameIsInvalidText() {
        if (this.usernameBlurred || this.formSubmitted) {
          if (this.username === '') {
            return this.$tr('required');
          }
        }
        return '';
      },
      usernameIsInvalid() {
        return Boolean(this.usernameIsInvalidText);
      },
      passwordIsInvalidText() {
        if (this.passwordBlurred || this.formSubmitted) {
          if (this.simpleSignIn && this.password === '') {
            return this.$tr('requiredForCoachesAdmins');
          } else if (this.password === '') {
            return this.$tr('required');
          }
        }
        return '';
      },
      passwordIsInvalid() {
        // prevent validation from showing when we only think that the password is empty
        if (this.autoFilledByChromeAndNotEdited) {
          return false;
        }
        return Boolean(this.passwordIsInvalidText);
      },
      formIsValid() {
        if (this.simpleSignIn) {
          return !this.usernameIsInvalid;
        }
        return !this.usernameIsInvalid && !this.passwordIsInvalid;
      },
      canSignUp() {
        return this.facilityConfig.learnerCanSignUp;
      },
      signUpPage() {
        return { name: PageNames.SIGN_UP };
      },
      versionMsg() {
        return this.$tr('poweredBy', { version: __version });
      },
      hasServerError() {
        return Boolean(this.passwordMissing || this.invalidCredentials);
      },
      needPasswordField() {
        return !this.simpleSignIn || this.hasServerError;
      },
      allowGuestAccess() {
        return this.facilityConfig.allowGuestAccess;
      },
      logoHeight() {
        const CRITICAL_ACTIONS_HEIGHT = 350; // title + form + action buttons
        let height = this.windowHeight - CRITICAL_ACTIONS_HEIGHT - 32;
        height = Math.max(height, 32);
        height = Math.min(height, 80);
        return height;
      },
      logoTextSize() {
        return Math.floor(this.logoHeight * 0.3);
      },
      guestURL() {
        return urls['kolibri:guest']();
      },
    },
    watch: {
      username: 'setSuggestionTerm',
    },
    mounted() {
      /*
        Chrome has non-standard behavior with auto-filled text fields where
        the value shows up as an empty string even though there is text in
        the field:
          https://bugs.chromium.org/p/chromium/issues/detail?id=669724
        As super-brittle hack to detect the presence of auto-filled text and
        work-around it, we look for a change in background color as described
        here:
          https://stackoverflow.com/a/35783761
      */
      setTimeout(() => {
        const bgColor = window.getComputedStyle(this.$refs.username.$el.querySelector('input'))
          .backgroundColor;

        if (bgColor === 'rgb(250, 255, 189)') {
          this.autoFilledByChromeAndNotEdited = true;
        }
      }, 250);
    },
    methods: {
      ...mapActions(['kolibriLogin']),
      closeFacilityModal() {
        this.facilityModalVisible = false;
      },
      setSuggestionTerm(newVal) {
        if (newVal !== null && typeof newVal !== 'undefined') {
          // Only check if defined or not null
          if (newVal.length < 3) {
            // Don't search for suggestions if less than 3 characters entered
            this.suggestionTerm = '';
            this.usernameSuggestions = [];
          } else if (
            (!newVal.startsWith(this.suggestionTerm) && this.suggestionTerm.length) ||
            !this.suggestionTerm.length
          ) {
            // We have already set a suggestion search term
            // The currently set suggestion term does not match the current username
            // Or we do not currently have a suggestion term set
            // Set it to the new term and fetch new suggestions
            this.suggestionTerm = newVal;
            this.setSuggestions();
          }
        }
      },
      setSuggestions() {
        FacilityUsernameResource.fetchCollection({
          getParams: {
            facility: this.facility,
            search: this.suggestionTerm,
          },
        })
          .then(users => {
            this.usernameSuggestions = users.map(user => user.username);
            this.showDropdown = true;
          })
          .catch(() => {
            this.usernameSuggestions = [];
          });
      },
      handleKeyboardNav(e) {
        switch (e.code) {
          case 'ArrowDown':
            if (this.showDropdown && this.suggestions.length) {
              this.highlightedIndex = Math.min(
                this.highlightedIndex + 1,
                this.suggestions.length - 1
              );
            }
            break;
          case 'ArrowUp':
            if (this.showDropdown && this.suggestions.length) {
              this.highlightedIndex = Math.max(this.highlightedIndex - 1, -1);
            }
            break;
          case 'Escape':
            this.showDropdown = false;
            break;
          case 'Enter':
            if (this.highlightedIndex < 0) {
              this.showDropdown = false;
            } else {
              this.fillUsername(this.suggestions[this.highlightedIndex]);
              e.preventDefault();
            }
            break;
          default:
            this.showDropdown = true;
        }
      },
      fillUsername(username) {
        // Only do this if we have been passed a non-null value
        if (username !== null && typeof username !== 'undefined') {
          this.username = username;
          this.showDropdown = false;
          this.highlightedIndex = -1;
          // focus on input after selection
          this.$refs.username.focus();
        }
      },
      handleUsernameBlur() {
        this.usernameBlurred = true;
        this.showDropdown = false;
      },
      signIn() {
        this.formSubmitted = true;
        if (this.formIsValid) {
          this.kolibriLogin({
            username: this.username,
            password: this.password,
            facility: this.facilityId,
          }).catch();
        } else {
          this.focusOnInvalidField();
        }
      },
      focusOnInvalidField() {
        if (this.usernameIsInvalid) {
          this.$refs.username.focus();
        } else if (this.passwordIsInvalid) {
          this.$refs.password.focus();
        }
      },
      handlePasswordChanged() {
        this.autoFilledByChromeAndNotEdited = false;
      },
    },
  };

</script>


<style lang="scss" scoped>

  @import '~kolibri.styles.definitions';

  .fh {
    height: 100%;
  }

  .wrapper-table {
    display: table;
    width: 100%;
    height: 100%;
    text-align: center;
  }

  .main-row {
    // Workaround for print-width css issue https://github.com/prettier/prettier/issues/4460
    $bk-img: linear-gradient(rgba(0, 0, 0, 0.7), rgba(0, 0, 0, 0.7)), url('./background.jpg');

    display: table-row;
    text-align: center;
    background-color: $core-action-normal;
    background-image: $bk-img;
    background-repeat: no-repeat;
    background-position: center;
    background-size: cover;
  }

  .main-cell {
    display: table-cell;
    height: 100%;
    vertical-align: middle;
  }

  .box {
    width: 300px;
    margin: 16px auto;
    background-color: $core-bg-light;
    box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.2), 0 1px 1px 0 rgba(0, 0, 0, 0.14),
      0 2px 1px -1px rgba(0, 0, 0, 0.12);
  }

  .logo {
    margin-top: 16px;
  }

  .login-form {
    position: relative;
    width: 70%;
    max-width: 300px;
    margin: auto;
    text-align: left;
  }

  .login-btn {
    width: calc(100% - 16px);
  }

  .version {
    padding-bottom: 16px;
    margin-top: 24px;
    margin-bottom: 0;
    font-size: 0.8em;
  }

  .footer-row {
    display: table-row;
    background-color: $core-bg-light;
  }

  .footer-cell {
    display: table-cell;
    min-height: 56px;
    padding: 16px;
    vertical-align: middle;
  }

  .suggestions {
    position: absolute;
    z-index: 8;
    width: 100%;
    padding: 0;
    margin: 0;
    // Move up snug against the textbox
    margin-top: -2em;
    list-style-type: none;
    background-color: white;
    box-shadow: 1px 2px 8px #e6e6e6;
  }

  .highlighted {
    background-color: $core-grey;
  }

  .textbox-enter-active {
    transition: opacity 0.5s;
  }

  .textbox-enter {
    opacity: 0;
  }

  .list-leave-active {
    transition: opacity 0.1s;
  }

  .textbox-leave {
    transition: opacity 0s;
  }

  h1 {
    margin-top: 0;
    font-size: 1.5em;
    font-weight: 100;
    color: #9174a9;
  }

  .create-button {
    margin-top: 16px;
  }

  .guest-button {
    font-size: 14px;
  }

</style>