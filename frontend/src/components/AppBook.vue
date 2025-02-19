<template>
  <NModal v-model:show="showModel" @close="hideInfo">
    <NCard hoverable :class="$style.card">
      <NThing content-indented>
        <template #header>
          <NP :class="[$style.max70vw, $style.title]">
            {{ info.title }}
          </NP>
        </template>
        <template #header-extra> # {{ info.number }} </template>
        <template #avatar>
          <NImage
            :width="imgWidth"
            :src="info.img || '/defaultBook.png'"
            :preview-disabled="!info.img"
          />
        </template>
        <template #description>{{ info.author }}</template>
        <NP :class="$style.max70vw">
          {{ info.desc }}
        </NP>
        <template #footer>
          <NSpace>
            <NButton
              v-for="(tag, index) in info.tags"
              :key="info.id + index.toString()"
              round
              secondary
              size="small"
            >
              {{ tag }}
            </NButton>
          </NSpace>
        </template>
        <template #action>
          <NP :class="$style.max70vw"> 状态：{{ bookStatus }} </NP>
        </template>
      </NThing>
      <template #action>
        <NSpace>
          <NButton type="error" @click="hideInfo">关闭</NButton>
          <NPopconfirm
            v-if="authState.user.role === 1 && info.status === 0"
            :positive-text="info.number === 0 ? '确认已接收' : '确认撤回接收'"
            negative-text="取消"
            @positive-click="handleConfirm"
          >
            <template #trigger>
              <NButton v-if="info.number === 0" type="primary">
                已接收
                <template #icon>
                  <NIcon>
                    <CheckmarkDoneOutline />
                  </NIcon>
                </template>
              </NButton>
              <NButton v-else type="primary">
                取消接收
                <template #icon>
                  <NIcon>
                    <ArrowUndoSharp />
                  </NIcon>
                </template>
              </NButton>
            </template>
            确定？
          </NPopconfirm>
          <NPopconfirm
            v-if="
              (authState.user.role === 1 && [2, 3].includes(info.status)) ||
              (info.status === 2 && info.orderBy?.id === authState.user.id)
            "
            :positive-text="info.status === 2 ? '确认已领取' : '确认撤回领取'"
            negative-text="取消"
            @positive-click="handleGive"
          >
            <template #trigger>
              <NButton v-if="info.status === 2" type="primary">
                已领取
                <template #icon>
                  <NIcon>
                    <CheckmarkDoneOutline />
                  </NIcon>
                </template>
              </NButton>
              <NButton v-else type="primary">
                取消已领取
                <template #icon>
                  <NIcon>
                    <ArrowUndoSharp />
                  </NIcon>
                </template>
              </NButton>
            </template>
            确定？
          </NPopconfirm>
          <NPopconfirm
            v-if="
              (info.owner.id === authState.user.id && info.status === 0) ||
              authState.user.role === 1
            "
            positive-text="确认"
            negative-text="取消"
            @positive-click="handleDelete"
          >
            <template #trigger>
              <NButton type="error">
                删除
                <template #icon>
                  <NIcon>
                    <TrashOutline />
                  </NIcon>
                </template>
              </NButton>
            </template>
            确定？
          </NPopconfirm>
          <NPopconfirm
            v-if="
              (info.status === 1 && info.owner.id !== authState.user.id) ||
              (info.status === 2 && info.orderBy)
            "
            positive-text="确认"
            negative-text="取消"
            @positive-click="handlePatch"
          >
            <template #trigger>
              <NButton
                v-if="info.status === 1 && info.owner.id !== authState.user.id"
                type="primary"
              >
                预定
                <template #icon>
                  <NIcon>
                    <CheckmarkDoneOutline />
                  </NIcon>
                </template>
              </NButton>
              <NButton v-if="info.status === 2 && info.orderBy" type="warning">
                取消预定
                <template #icon>
                  <NIcon>
                    <ReturnUpBackOutline />
                  </NIcon>
                </template>
              </NButton>
            </template>
            确定？
          </NPopconfirm>
        </NSpace>
      </template>
    </NCard>
  </NModal>
</template>
<script setup lang="ts">
import {
  CheckmarkDoneOutline,
  ReturnUpBackOutline,
  TrashOutline,
  ArrowUndoSharp,
} from '@vicons/ionicons5';
import {
  NButton,
  NCard,
  NIcon,
  NImage,
  NSpace,
  NThing,
  NPopconfirm,
  useMessage,
  NModal,
  NP,
} from 'naive-ui';
import { ref, watch } from 'vue';
import getStatus from '../utils/getStatus';
import IFrontendBook from '../../../types/IFrontendBook';
import deleteBook from '../service/deleteBook';
import getMe from '../service/getMe';
import orderBook from '../service/orderBook';
import putBook from '../service/putBook';
import receiveBook from '../service/receiveBook';
import useAuthStore from '../stores/authState';
import useBooksStore from '../stores/booksState';
import useLoadingStore from '../stores/loadingState';
import giveOutBook from '../service/giveOutBook';

const imgWidth = window.innerWidth * 0.08 < 50 ? 50 : window.innerWidth * 0.08;
const initInfo: IFrontendBook = {
  id: '',
  desc: '',
  title: '',
  author: '',
  owner: { id: '', avatar: null },
  tags: [],
  img: '',
  status: 1,
  number: 0,
};
const info = ref(initInfo);
const authState = useAuthStore();
const bookState = useBooksStore();
const loadState = useLoadingStore();
const message = useMessage();
const bookStatus = ref(getStatus(info.value));
watch(info, () => {
  bookStatus.value = getStatus(info.value);
});
const showModel = ref(false);
const hideInfo = () => {
  showModel.value = false;
};
const popInfo = (newInfo: IFrontendBook) => {
  info.value = newInfo;
  showModel.value = true;
};
const handlePatch = async () => {
  if (!authState.authed) return message.error('请先登录');
  const res = await orderBook(info.value);
  if (!res) return message.error('操作失败');
  const me = await getMe();
  authState.$patch({ user: me });
  if (res.orderBy) {
    message.success(`预定成功，还可预定 ${me.committedBooks - me.orderedBooks} 本`);
  } else {
    message.warning(`取消成功，还可预定 ${me.committedBooks - me.orderedBooks} 本`);
  }
  bookState.update(res);
  return hideInfo();
};
const handleConfirm = async () => {
  const handleReceive = async () => {
    if (!authState.authed || authState.user.role !== 1) return message.error('无权限');
    loadState.loading = true;
    const book = await receiveBook(info.value.id);
    info.value = book;
    loadState.loading = false;
    message.success(`已确认收到，编号：${book.number}`);
    return bookState.update(book);
  };
  const handleUnReceive = async () => {
    if (!authState.authed || authState.user.role !== 1) return message.error('无权限');
    loadState.loading = true;
    const book = await putBook(info.value.id, { number: 0 });
    info.value = book;
    loadState.loading = false;
    message.success('已取消');
    return bookState.update(book);
  };
  if (info.value.number === 0) {
    await handleReceive();
  } else {
    await handleUnReceive();
  }
};

const handleGive = async () => {
  if (!authState.authed) return message.error('无权限');
  const handleGiveOut = async () => {
    if (authState.user.role !== 1 && info.value.orderBy?.id !== authState.user.id)
      return message.error('无权限');
    loadState.loading = true;
    let book;
    if (authState.user.role === 1) {
      book = await putBook(info.value.id, { status: 3 });
    } else {
      book = await giveOutBook(info.value.id);
    }
    info.value = book;
    loadState.loading = false;
    message.success('已确认给出');
    return bookState.update(book);
  };
  const handleUnGiveOut = async () => {
    if (authState.user.role !== 1) return message.error('无权限');
    loadState.loading = true;
    const book = await putBook(info.value.id, { status: 2 });
    info.value = book;
    loadState.loading = false;
    message.success('已取消');
    return bookState.update(book);
  };
  if (info.value.status === 2) {
    await handleGiveOut();
  } else {
    await handleUnGiveOut();
  }
  return null;
};

const handleDelete = async () => {
  loadState.loading = true;
  try {
    const res = await deleteBook(info.value.id);
    switch (res.status) {
      case 204:
        message.success('删除成功');
        break;
      case 404:
        message.warning('书本已不存在，可能是重复删除');
        break;
      case 401:
        message.error('无权限');
        break;
      default:
        message.error('未知错误');
        break;
    }
    bookState.delete(info.value.id);
  } catch (e) {
    message.error('删除失败');
    console.error(e);
  }
  hideInfo();
  loadState.loading = false;
};
defineExpose({ popInfo, hideInfo });
</script>
<style module>
.max70vw {
  max-width: 70vw;
  word-break: break-all;
}
.title {
  font-size: 16px;
}
.card {
  max-width: 80rem;
}
</style>
