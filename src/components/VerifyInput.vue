<script setup lang="ts">
import type { VerifyData, VerifyPayload, VerifyResult } from '@/types'
import { decode as base2048Decode } from '@/utils/base2048'
import { decode as unpack } from 'msgpack-lite'
import dayjs from 'dayjs'
import relativeTime from 'dayjs/plugin/relativeTime'
import 'dayjs/locale/zh-cn'
import { KJUR } from 'jsrsasign'
import publicKey from '../assets/seal_trusted.public.pem?raw'

dayjs.locale('zh-cn')
dayjs.extend(relativeTime)

const sealCode = ref<string>('')
const result = ref<VerifyResult>()
const time = computed(() => {
  return result.value?.success ? dayjs.unix(result.value.timestamp) : undefined
})

function toHexString(byteArray: Uint8Array) {
  return Array.from(byteArray, (b) => ('0' + (b & 0xff).toString(16)).slice(-2)).join('')
}

const signVerify = (payload: Uint8Array, signData: Uint8Array): boolean => {
  const data = toHexString(payload)
  const sign = toHexString(signData)

  const sig = new KJUR.crypto.Signature({ alg: 'SHA1withECDSA' })
  sig.init(publicKey.trim())
  sig.updateHex(data)

  return sig.verify(sign)
}

const verify = async () => {
  if (!sealCode.value || sealCode.value.length === 0) {
    return
  }
  if (!sealCode.value.startsWith('SEAL#') && !sealCode.value.startsWith('SEAL%')) {
    result.value = {
      success: false,
      err: '不是可识别的海豹码'
    }
    return
  }

  // 解析
  const code = sealCode.value.slice('SEAL'.length)
  let data: VerifyData
  try {
    let rawData: Uint8Array
    if (code.startsWith('#')) {
      // base64 编码的海豹校验码
      rawData = Uint8Array.from(atob(code.slice(1)), (c) => c.charCodeAt(0))
    } else {
      // base2048 编码的海豹校验码
      rawData = base2048Decode(code.slice(1))
    }
    data = unpack(rawData)
  } catch (e) {
    console.error('parse error', e)
    result.value = {
      success: false,
      err: '无法解析海豹码，是否复制完全？'
    }
    return
  }

  console.log('parsed', data)
  // 校验
  if (!data.sign || data.sign.length === 0) {
    result.value = {
      success: false,
      err: '该海豹码不是官方发布的海豹生成的！'
    }
    return
  } else {
    const isValid = signVerify(data.payload, data.sign)
    if (!isValid) {
      result.value = {
        success: false,
        err: '该海豹码不是官方发布的海豹生成的！'
      }
      return
    }
  }

  const payload: VerifyPayload = unpack(data.payload)
  result.value = {
    success: true,
    ...payload
  }
}

const getTimeDiffLevel = (t: dayjs.Dayjs) => {
  const now = dayjs()
  const diff = Math.abs(now.diff(t, 'minute'))
  if (diff >= 24 * 60) {
    return 2
  } else if (diff >= 5) {
    return 1
  }
  return 0
}

const getTimeTagType = (t: dayjs.Dayjs) => {
  const diffLevel = getTimeDiffLevel(t)
  if (diffLevel >= 2) {
    return 'error'
  } else if (diffLevel >= 1) {
    return 'warning'
  }
  return 'success'
}
</script>

<template>
  <div class="break-all">
    <n-input
      type="textarea"
      :autosize="{ minRows: 3 }"
      placeholder="请输入生成的神秘海豹码，以「SEAL%」或「SEAL#」开头"
      v-model:value="sealCode"
      @blur="verify"
    />
  </div>

  <n-flex v-if="result" vertical align="center" class="mt-8 text-base">
    <template v-if="result.success">
      <n-text
        :type="getTimeDiffLevel(time!) >= 2 ? 'warning' : 'success'"
        class="my-4 flex items-center gap-2 text-xl"
      >
        <n-icon size="32"><i-carbon-checkmark-filled /></n-icon>
        校验通过
        <span v-if="getTimeDiffLevel(time!) >= 2">，但……</span>
      </n-text>

      <div class="max-w-xl">
        <n-text class="break-all"
          >由 &lt;{{ result.username }}&gt;({{ result.uid }}) 于 {{ result.platform }} 生成</n-text
        >
      </div>

      <n-flex v-if="time" justify="center" align="center">
        <n-text :type="getTimeTagType(time)">
          生成时间：{{ time.format('YYYY-MM-DD HH:mm:ss') }}
        </n-text>
        <n-tag :bordered="false" :type="getTimeTagType(time)">{{ time.fromNow() }}</n-tag>
      </n-flex>

      <n-flex justify="center" align="center">
        <n-text>海豹版本为</n-text>
        <n-tag :bordered="false" type="info">{{ result.version }}</n-tag>
      </n-flex>
    </template>
    <template v-else>
      <n-text type="error" class="flex items-center gap-2 text-lg">
        <n-icon size="32"><i-carbon-error-filled /></n-icon>
        校验失败！{{ result.err }}
      </n-text>
    </template>
  </n-flex>
</template>

<style scoped></style>
