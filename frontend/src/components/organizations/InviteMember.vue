<template>
  <div class="InviteMember">
    <!-- org detail invite button -->
    <button
      v-if="role === 'admin'"
      @click="dialogVisible = true"
      class="btn btn-secondary-gray btn-md"
    >
      <SvgIcon name="invite_org_member"/>
      <span>{{ $t('organization.invite.title') }}</span>
    </button>
    <!-- invite dialog -->
    <el-dialog
      v-model="dialogVisible"
      top="10vh"
      :style="{ borderRadius: '10px' }"
      width="450"
      class="invite_dialog"
    >
      <template #header>
        <div class="flex justify-between">
          <div
            class="px-[12px] py-[12px] rounded-lg border-[2px] border-gray-200"
          >
            <SvgIcon name="invite_org_member" />
          </div>
          <img
            src="/images/invite_bg.png"
            class="w-[200px] absolute top-0 left-0"
          />
        </div>
      </template>
      <!-- dialog content -->
      <div class="relative">
        <div class="text-lg leading-[28px] text-gray-900">
          {{ $t('organization.invite.inviteNew') }}
        </div>
        <span class="text-sm leading-[20px] text-gray-600 font-light"
          >{{ $t('organization.invite.inviteDesc') }}
          <span class="font-bold">{{ orgName }}</span></span
        >
        <div class="mt-[20px]">
          <div>
            <div class="mb-[20px]">
              <p class="text-gray-700 text-sm mb-[6px]">
                {{ $t('all.role') }}
              </p>
              <el-select
                v-model="userRoleInput"
                :placeholder="$t('all.select')"
                size="large"
                class="w-full"
              >
                <el-option
                  v-for="item in roleMappings"
                  :key="item.value"
                  :label="item.label"
                  :value="item.value"
                />
              </el-select>
            </div>
            <p class="text-gray-700 text-sm mb-[6px]">
              {{ $t('all.userName') }}
            </p>
            <div
              class="flex gap-[4px] flex-wrap items-center w-full border rounded-xs border-gray-300 min-h-[40px] p-[6px]"
            >
              <div
                class="scroll-container flex gap-[4px] flex-wrap max-h-[120px] overflow-y-auto"
              >
                <span
                  v-for="user in selectedUsers"
                  class="flex items-center gap-[5px] border rounded-xs border-gray-300 px-[5px] py-[2px]"
                >
                  <img
                    :src="user.avatar"
                    height="16"
                    width="16"
                  />
                  {{ user.username }}
                  <el-icon><Close @click="removeUser(user.username)" /></el-icon>
                </span>
              </div>
              <input
                class="w-full max-h-[36px] outline-none"
                v-model="userNameInput"
                @input="showUserList"
              />
            </div>
            <div
              v-show="shouldShowUserList"
              class="md:max-h-[110px] max-h-[210px] overflow-y-auto rounded-md border border-gray-200 bg-white shadow-lg py-[4px] px-[6px]"
            >
              <p
                v-for="user in userList"
                @click="selectUser(user)"
                class="flex gap-[8px] items-center  p-[10px]"
                :class="user.invited ? 'cursor-not-allowed' : 'cursor-pointer'"
              >
                <img  
                  :src="user.avatar || 'https://cdn.casbin.org/img/casbin.svg'"
                  height="16"
                  width="16"
                />
                {{ user.username }}
                <span>{{ user.invited ? $t('organization.invite.invited') : '' }}</span>
              </p>
            </div>
          </div>
        </div>
      </div>
      <template #footer>
        <span class="flex justify-between px-4 gap-2">
          <button
            class="btn btn-secondary-gray btn-md flex-1"
            @click="dialogVisible = false"
            >{{ $t('all.cancel') }}</button
          >
          <button
            class="btn btn-primary btn-md flex-1"
            @click="confirmInviteNewMember"
          >
            {{ $t('all.confirm') }}
          </button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
  import { ref, onMounted } from 'vue'
  import useFetchApi from '../../packs/useFetchApi'
  import { ElMessage } from 'element-plus'
  import { useI18n } from 'vue-i18n'

  const emit = defineEmits(['resetMemberList'])

  const props = defineProps({
    orgName: String,
    role: String
  })

  const { t } = useI18n()
  const dialogVisible = ref(false)
  const userNameInput = ref('')
  const userRoleInput = ref('read')
  const selectedUsers = ref([])
  const membersList = ref([])
  const userList = ref([])
  const shouldShowUserList = ref(false)
  const roleMappings = [
    {
      value: 'read',
      label: 'read'
    },
    {
      value: 'write',
      label: 'write'
    },
    {
      value: 'admin',
      label: 'admin'
    }
  ]

  const removeUser = (username) => {
    selectedUsers.value = selectedUsers.value.filter(
      (item) => item.username !== username
    )
  }

  const fetchOrgMemberList = async () => {
    const orgMemberListEndpoint = `/organization/${props.orgName}/members`
    const { data, error } = await useFetchApi(orgMemberListEndpoint).json()
    if (error.value) {
      ElMessage({ message: error.value.msg, type: 'warning' })
    } else {
      const body = data.value
      membersList.value = body.data.data
    }
  }

  const selectUser = (newUser) => {
    if(newUser.invited){
      return
    }
    const findUser = selectedUsers.value.find(
      (user) => user.username === newUser.username
    )
    if (!findUser) {
      selectedUsers.value.push({ username: newUser.username, avatar: newUser.avatar || 'https://cdn.casbin.org/img/casbin.svg' })
    }
    userNameInput.value = ''
    shouldShowUserList.value = false
  }

  const showUserList = (e) => {
    if (e.target.value) {
      getUsers(userNameInput.value)
    } else {
      shouldShowUserList.value = false
    }
  }

  async function getUsers(username) {
    const usersEndpoint = `/users?search=${username}`
    const options = {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json'
      }
    }
    const {data, error} = await useFetchApi(usersEndpoint, options).json()
    if (data.value) {
      shouldShowUserList.value = data.value.data.total > 0
      // Add invited
      userList.value = data.value.data.data.slice(0, 6).map(user => ({
        ...user,
        invited: membersList.value.find(member => member.username === user.username) !== undefined
      }));
    } else {
      ElMessage.warning(error.value.msg)
    }
  }

  const confirmInviteNewMember = () => {
    inviteNewMember()
      .then(() => {
        emit('resetMemberList', selectedUsers.value, userRoleInput.value)
        selectedUsers.value = []
        fetchOrgMemberList()
        dialogVisible.value = false
      })
      .catch((err) => {
        ElMessage({
          message: err.message,
          type: 'warning'
        })
      })
  }

  async function inviteNewMember() {
    const inviteNewMemberEndpoint = `/organization/${props.orgName}/members`
    const options = {
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        role: userRoleInput.value,
        users: selectedUsers.value.map((user) => user.username).join(',')
      })
    }
    const { error } = await useFetchApi(inviteNewMemberEndpoint, options).post().json()
    if (error.value) {
      ElMessage({ message: error.value.msg, type: 'warning' })
    } else {
      ElMessage({
        message: t('organization.invite.addSuccess'),
        type: 'success'
      })
      return true
    }
  }
  onMounted(() => {
    fetchOrgMemberList()
  })
</script>

<style>
  @media (max-width: 768px) {
    .InviteMember .invite_dialog {
      width: 350px;
    }
  }
  .InviteMember .scroll-container::after {
    content: '';
    position: absolute;
    top: 0;
    right: 0;
    width: 8px; /* 滚动条宽度 */
    background-color: #ccc; /* 滚动条颜色 */
    border-radius: var(--border-radius-xs); /* 滚动条圆角 */
  }

  .InviteMember .scroll-container .content {
    padding-right: 8px; /* 留出滚动条的空间 */
  }

  .InviteMember .scroll-container::-webkit-scrollbar {
    width: 8px; /* 滚动条宽度 */
  }

  .InviteMember .scroll-container::-webkit-scrollbar-thumb {
    background-color: #888; /* 滚动条thumb颜色 */
    border-radius: var(--border-radius-xs); /* 滚动条thumb圆角 */
  }
</style>
