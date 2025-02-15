<template>
  <div class="flex items-center w-full overflow-y-auto">
    <div class="flex flex-col h-full p-5 pt-16 mx-auto my-0 font-inter">
      <div class="flex flex-col gap-16 sm:max-w-[720px]">
        <div class="flex flex-col gap-6">
          <h2 class="mt-4 text-2xl font-medium text-ash-900">
            {{ $t('PROFILE_SETTINGS.TITLE') }}
          </h2>
          <user-profile-picture
            :src="avatarUrl"
            :name="name"
            size="72px"
            @change="updateProfilePicture"
            @delete="deleteProfilePicture"
          />
          <user-basic-details
            :name="name"
            :display-name="displayName"
            :email="email"
            :email-enabled="!globalConfig.disableUserProfileUpdate"
            @update-user="updateProfile"
          />
        </div>

        <form-section
          :title="$t('PROFILE_SETTINGS.FORM.MESSAGE_SIGNATURE_SECTION.TITLE')"
          :description="
            $t('PROFILE_SETTINGS.FORM.MESSAGE_SIGNATURE_SECTION.NOTE')
          "
        >
          <message-signature
            :message-signature="messageSignature"
            @update-signature="updateSignature"
          />
        </form-section>
        <form-section
          :header="$t('PROFILE_SETTINGS.FORM.SEND_MESSAGE.TITLE')"
          :description="$t('PROFILE_SETTINGS.FORM.SEND_MESSAGE.NOTE')"
        >
          <div
            class="flex flex-col justify-between w-full gap-5 sm:gap-4 sm:flex-row"
          >
            <button
              v-for="hotKey in hotKeys"
              :key="hotKey.key"
              class="px-0 reset-base"
            >
              <hot-key-card
                :key="hotKey.title"
                :title="hotKey.title"
                :description="hotKey.description"
                :light-image="hotKey.lightImage"
                :dark-image="hotKey.darkImage"
                :active="isEditorHotKeyEnabled(uiSettings, hotKey.key)"
                @click="toggleHotKey(hotKey.key)"
              />
            </button>
          </div>
        </form-section>
        <form-section
          :header="$t('PROFILE_SETTINGS.FORM.PASSWORD_SECTION.TITLE')"
        >
          <change-password v-if="!globalConfig.disableUserProfileUpdate" />
        </form-section>
      </div>
    </div>
  </div>
</template>
<script>
import globalConfigMixin from 'shared/mixins/globalConfigMixin';
import uiSettingsMixin, {
  isEditorHotKeyEnabled,
} from 'dashboard/mixins/uiSettings';
import alertMixin from 'shared/mixins/alertMixin';
import { mapGetters } from 'vuex';
import { clearCookiesOnLogout } from 'dashboard/store/utils/api.js';

import UserProfilePicture from './UserProfilePicture.vue';
import UserBasicDetails from './UserBasicDetails.vue';
import MessageSignature from './MessageSignature.vue';
import HotKeyCard from './HotKeyCard.vue';
import ChangePassword from './ChangePassword.vue';
import FormSection from 'dashboard/components/FormSection.vue';

export default {
  components: {
    MessageSignature,
    FormSection,
    UserProfilePicture,
    UserBasicDetails,
    HotKeyCard,
    ChangePassword,
  },
  mixins: [alertMixin, globalConfigMixin, uiSettingsMixin],
  data() {
    return {
      avatarFile: '',
      avatarUrl: '',
      name: '',
      displayName: '',
      email: '',
      messageSignature: '',
      hotKeys: [
        {
          key: 'enter',
          title: this.$t(
            'PROFILE_SETTINGS.FORM.SEND_MESSAGE.CARD.ENTER_KEY.HEADING'
          ),
          description: this.$t(
            'PROFILE_SETTINGS.FORM.SEND_MESSAGE.CARD.ENTER_KEY.CONTENT'
          ),
          lightImage: '/assets/images/dashboard/profile/hot-key-enter.svg',
          darkImage: '/assets/images/dashboard/profile/hot-key-enter-dark.svg',
        },
        {
          key: 'cmd_enter',
          title: this.$t(
            'PROFILE_SETTINGS.FORM.SEND_MESSAGE.CARD.CMD_ENTER_KEY.HEADING'
          ),
          description: this.$t(
            'PROFILE_SETTINGS.FORM.SEND_MESSAGE.CARD.CMD_ENTER_KEY.CONTENT'
          ),
          lightImage: '/assets/images/dashboard/profile/hot-key-ctrl-enter.svg',
          darkImage:
            '/assets/images/dashboard/profile/hot-key-ctrl-enter-dark.svg',
        },
      ],
    };
  },
  computed: {
    ...mapGetters({
      currentUser: 'getCurrentUser',
      currentUserId: 'getCurrentUserID',
      globalConfig: 'globalConfig/get',
    }),
  },
  mounted() {
    if (this.currentUserId) {
      this.initializeUser();
    }
  },
  methods: {
    initializeUser() {
      this.name = this.currentUser.name;
      this.email = this.currentUser.email;
      this.avatarUrl = this.currentUser.avatar_url;
      this.displayName = this.currentUser.display_name;
      this.messageSignature = this.currentUser.message_signature;
    },
    isEditorHotKeyEnabled,
    async dispatchUpdate(payload, successMessage, errorMessage) {
      let alertMessage = '';
      try {
        await this.$store.dispatch('updateProfile', payload);
        alertMessage = successMessage;

        return true; // return the value so that the status can be known
      } catch (error) {
        alertMessage = error?.response?.data?.error
          ? error.response.data.error
          : errorMessage;

        return false; // return the value so that the status can be known
      } finally {
        this.showAlert(alertMessage);
      }
    },
    async updateProfile(userAttributes) {
      const { name, email, displayName } = userAttributes;
      const hasEmailChanged = this.currentUser.email !== email;
      this.name = name || this.name;
      this.email = email || this.email;
      this.displayName = displayName || this.displayName;

      const updatePayload = {
        name: this.name,
        email: this.email,
        displayName: this.displayName,
        avatar: this.avatarFile,
      };

      const success = await this.dispatchUpdate(
        updatePayload,
        hasEmailChanged
          ? this.$t('PROFILE_SETTINGS.AFTER_EMAIL_CHANGED')
          : this.$t('PROFILE_SETTINGS.UPDATE_SUCCESS'),
        this.$t('RESET_PASSWORD.API.ERROR_MESSAGE')
      );

      if (hasEmailChanged && success) clearCookiesOnLogout();
    },
    async updateSignature(signature) {
      const payload = { message_signature: signature };
      let successMessage = this.$t(
        'PROFILE_SETTINGS.FORM.MESSAGE_SIGNATURE_SECTION.API_SUCCESS'
      );
      let errorMessage = this.$t(
        'PROFILE_SETTINGS.FORM.MESSAGE_SIGNATURE_SECTION.API_ERROR'
      );

      await this.dispatchUpdate(payload, successMessage, errorMessage);
    },
    updateProfilePicture({ file, url }) {
      this.avatarFile = file;
      this.avatarUrl = url;
    },
    async deleteProfilePicture() {
      try {
        await this.$store.dispatch('deleteAvatar');
        this.avatarUrl = '';
        this.avatarFile = '';
        this.showAlert(this.$t('PROFILE_SETTINGS.AVATAR_DELETE_SUCCESS'));
      } catch (error) {
        this.showAlert(this.$t('PROFILE_SETTINGS.AVATAR_DELETE_FAILED'));
      }
    },
    toggleHotKey(key) {
      this.hotKeys = this.hotKeys.map(hotKey =>
        hotKey.key === key ? { ...hotKey, active: !hotKey.active } : hotKey
      );
      this.updateUISettings({ editor_message_key: key });
      this.showAlert(
        this.$t('PROFILE_SETTINGS.FORM.SEND_MESSAGE.UPDATE_SUCCESS')
      );
    },
  },
};
</script>
