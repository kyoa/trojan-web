<template>
  <div>
    <el-form :inline="true" label-width="80px" style="height: 29px;">
    <el-form-item size="mini">
        <el-button-group>
            <el-button type="primary" icon="el-icon-refresh" @click.native="refresh()">{{ $t('refresh') }}</el-button>
            <el-button type="primary" icon="el-icon-plus" @click.native="commonType=2;userInfo.username='';userInfo.password='';userVisible=true" v-if="isAdmin">{{ $t('add') }}</el-button>
            <el-button type="primary" icon="el-icon-refresh-left" @click.native="copySelection=multipleSelection;patchButton=true;commonType=1;confirmVisible=true" v-if="isAdmin">{{ $t('user.reset') }}</el-button>
            <el-button type="danger" icon="el-icon-delete" @click.native="copySelection=multipleSelection;patchButton=true;commonType=0;confirmVisible=true" v-if="isAdmin">{{ $t('delete') }}</el-button>
        </el-button-group>
    </el-form-item>
    </el-form>
    <el-table
    :data="dataList.filter(data => !search || data.Username.toLowerCase().includes(search.toLowerCase()))" style="width: 100%" :height="clientHeight" @selection-change="handleSelectionChange">
        <el-table-column
        type="selection"
        width="55" v-if="isAdmin">
        </el-table-column>
        <el-table-column
        :label="$t('username')"
        prop="Username"
        >
        </el-table-column>
        <el-table-column
        :label="$t('password')"
        :formatter="passwordFormatter">
        </el-table-column>
        <el-table-column
        :label="$t('user.upload')"
        :formatter="uploadFormatter" :sort-method="uploadSort" sortable>
        </el-table-column>
        <el-table-column
        :label="$t('user.download')"
        :formatter="downloadFormatter" :sort-method="downloadSort" sortable>
        </el-table-column>
        <el-table-column
        :label="$t('user.total')"
        :formatter="totalFormatter" :sort-method="totalSort" sortable>
        </el-table-column>
        <el-table-column
        :label="$t('user.quota')"
        :formatter="quotaFormatter">
        </el-table-column>
        <el-table-column
        width="170"
        align="center">
        <template slot="header">
            <el-input
                v-model="search"
                size="small"
                :placeholder="$t('user.search')" v-if="isAdmin"/>
            <div v-if="!isAdmin">{{ $t('user.operate') }}</div>
        </template>
        <template slot-scope="scope">
            <el-dropdown  size="mini" split-button type="text" v-if="isAdmin">
                {{ $t('edit') }}
                <el-dropdown-menu slot="dropdown">
                    <el-dropdown-item @click.native="userItem=scope.row; quotaVisible=true">{{ $t('user.limitData') }}</el-dropdown-item>
                    <el-dropdown-item @click.native="userItem=scope.row; commonType=1; patchButton=false; confirmVisible=true">{{ $t('user.reset') }}</el-dropdown-item>
                    <el-dropdown-item @click.native="userItem=scope.row; handelEditUser()">{{ $t('user.modifyUser') }}</el-dropdown-item>
                </el-dropdown-menu>
            </el-dropdown>
            <el-button
            size="mini"
            type="text"
            @click.native="userItem=scope.row;handleShare()"
            >{{ $t('share') }}</el-button>
            <el-button
            v-if="isAdmin"
            size="mini"
            type="text"
            @click.native="userItem=scope.row;commonType=0;patchButton=false;confirmVisible=true"
            >{{ $t('delete') }}</el-button>
        </template>
        </el-table-column>
    </el-table>
    <el-dialog :title="commonTitle" :visible.sync="userVisible" :width="dialogWidth">
        <el-input type="text" v-model="userInfo.username" :placeholder="$t('user.inputUsername')"/>
        <el-input type="text" v-model="userInfo.password" :placeholder="$t('user.inputPassword')" @keyup.enter.native="commonType === 2? handleAddUser(): handleUpdateUser()"/>
        <div slot="footer" class="dialog-footer">
            <el-button @click="userVisible = false">{{ $t('cancel') }}</el-button>
            <el-button type="primary" @click="commonType === 2? handleAddUser(): handleUpdateUser()">{{ $t('ok') }}</el-button>
        </div>
    </el-dialog>
    <el-dialog :title="commonTitle" :visible.sync="confirmVisible" :width="dialogWidth">
        {{ editUser }}
        <div slot="footer" class="dialog-footer">
            <el-button @click="confirmVisible = false;copySelection=[];">{{ $t('cancel') }}</el-button>
            <el-button type="primary" @click="confirmVisible = false; patchButton ? handlePatchOpera(): handleOpera()">{{ $t('ok') }}</el-button>
        </div>
    </el-dialog>
    <el-dialog :title="quotaText" :visible.sync="quotaVisible" :width="dialogWidth">
        <el-tooltip effect="dark" :content="$t('user.meanUnlimit')" placement="top">
            <el-input-number v-model="quota" :min="-1" :precision="0" size="mini"></el-input-number>
        </el-tooltip>
        <el-select v-model="quotaUnit" :placeholder="$t('choice')" size="mini" style="margin-left: 5px; width:80px">
            <el-option
            v-for="item in quotaOptions"
            :key="item.value"
            :label="item.value"
            :value="item.value">
            </el-option>
        </el-select>
        <div slot="footer" class="dialog-footer">
            <el-button @click="quotaVisible=false">{{ $t('cancel') }}</el-button>
            <el-button type="primary" @click="quotaVisible=false; handleSetQuota()">{{ $t('ok') }}</el-button>
        </div>
    </el-dialog>
    <el-dialog :title="$t('user.shareLink')" :visible.sync="qrcodeVisible" :width="dialogWidth" @close="closeQRCode">
        <div id="qrcode" ref="qrcode" class="qrcodeCenter"></div>
        <p class="qrcodeCenter"> {{ shareLink }} </p>
    </el-dialog>
  </div>
</template>

<script>
import { userList, addUser, delUser, updateUser } from '@/api/user'
import { setQuota, cleanData } from '@/api/data'
import { setDomain } from '@/api/trojan'
import { readablizeBytes, isValidIP } from '@/utils/common'
import { mapState } from 'vuex'
import QRCode from 'qrcodejs2'
export default {
    data() {
        return {
            search: '',
            domain: '',
            shareLink: '',
            dataList: [],
            multipleSelection: [],
            copySelection: [],
            clientHeight: 0,
            userVisible: false,
            confirmVisible: false,
            quotaVisible: false,
            qrcodeVisible: false,
            patchButton: false,
            // 确认框类型: 0删除, 1重置流量, 2新增用户, 3修改用户
            commonType: 0,
            userItem: null,
            quota: -1,
            quotaUnit: 'MB',
            quotaOptions: [
                {
                    value: 'MB'
                },
                {
                    value: 'GB'
                }
            ],
            userInfo: {
                username: '',
                password: ''
            }
        }
    },
    computed: {
        ...mapState(['dialogWidth', 'isAdmin']),
        commonTitle: function() {
            let text = ''
            if (this.commonType === 2) {
                text = this.$t('user.addUser')
            } else if (this.commonType === 3) {
                text = this.$t('user.modifyUser2') + this.userItem.Username + this.$t('user.userpass')
            } else if (this.commonType === 0) {
                if (this.patchButton) {
                    text = this.$t('user.patchDelUser')
                } else if (this.userItem !== null) {
                    text = this.$t('user.delUser') + this.userItem.Username + ' ?'
                }
            } else if (this.commonType === 1) {
                if (this.patchButton) {
                    text = this.$t('user.patchReset')
                } else if (this.userItem !== null) {
                    text = this.$t('user.resetUser') + this.userItem.Username + this.$t('user.data')
                }
            }
            return text
        },
        editUser: function() {
            if (this.patchButton) {
                let result = ''
                for (let i = 0; i < this.copySelection.length; i++) {
                    result += ', ' + this.copySelection[i].Username
                }
                return result.substring(1)
            } else {
                return ''
            }
        },
        quotaText: function() {
            if (this.userItem !== null) {
                return `${this.$t('user.limitUser')} ${this.userItem.Username} ${this.$t('user.data')}`
            } else {
                return ''
            }
        }
    },
    created() {
        this.refresh()
        this.clientHeight = document.body.clientHeight - 100
    },
    methods: {
        handleSelectionChange(val) {
            this.multipleSelection = val
        },
        passwordFormatter(row, column) {
            return atob(row.Password)
        },
        quotaFormatter(row, column) {
            return row.Quota === -1 ? this.$t('user.unlimit') : readablizeBytes(row.Quota)
        },
        uploadFormatter(row, column) {
            return readablizeBytes(row.Upload)
        },
        downloadFormatter(row, column) {
            return readablizeBytes(row.Download)
        },
        totalFormatter(row, column) {
            return readablizeBytes(row.Download + row.Upload)
        },
        uploadSort(a, b) {
            return a.Upload - b.Upload
        },
        downloadSort(a, b) {
            return a.Download - b.Download
        },
        totalSort(a, b) {
            return (a.Download + a.Upload) - (b.Download + b.Upload)
        },
        handleShare() {
            this.shareLink = `trojan://${atob(this.userItem.Password)}@${this.domain}:443`
            this.$nextTick(() => {
                this.createQRCode()
            })
            this.qrcodeVisible = true
        },
        handelEditUser() {
            this.userInfo.username = this.userItem.Username
            this.userInfo.password = atob(this.userItem.Password)
            this.commonType = 3
            this.userVisible = true
        },
        createQRCode() {
            // eslint-disable-next-line
            new QRCode('qrcode', {
                width: 200,
                height: 200,
                text: this.shareLink
            })
        },
        closeQRCode() {
            this.$refs.qrcode.innerHTML = ''
        },
        async handleSetQuota() {
            if (this.quota === -1) {
            } else if (this.quotaUnit === 'MB') {
                this.quota = this.quota * 1024 * 1024
            } else if (this.quotaUnit === 'GB') {
                this.quota = this.quota * 1024 * 1024 * 1024
            }
            const formData = new FormData()
            formData.set('id', this.userItem.ID)
            formData.set('quota', this.quota)
            const result = await setQuota(formData)
            if (result.Msg === 'success') {
                this.$message({
                    message: `${this.$t('user.limitUser2')}${this.userItem.Username}${this.$t('user.limitSuccess')}`,
                    type: 'success'
                })
                this.userItem = null
            } else {
                this.$message.error(result.Msg)
            }
            this.quota = 0
            this.refresh()
        },
        async handlePatchOpera() {
            let successText = ''
            let result = null
            for (let i = 0; i < this.copySelection.length; i++) {
                this.userItem = this.copySelection[i]
                if (this.commonType === 0) {
                    result = await delUser(this.userItem.ID)
                    successText = `${this.$t('user.delUser1')}${this.userItem.Username}${this.$t('user.success')}`
                } else if (this.commonType === 1) {
                    result = await cleanData(this.userItem.ID)
                    successText = `${this.$t('user.resetUser1')}${this.userItem.Username}${this.$t('user.success')}`
                }
                if (result.Msg === 'success') {
                    this.$message({
                        message: successText,
                        type: 'success'
                    })
                    this.userItem = null
                } else {
                    this.$message.error(result.Msg)
                }
            }
            this.refresh()
        },
        async handleOpera() {
            let successText = ''
            let result = null
            if (this.commonType === 0) {
                result = await delUser(this.userItem.ID)
                successText = `${this.$t('user.delUser1')}${this.userItem.Username}${this.$t('user.success')}`
            } else if (this.commonType === 1) {
                result = await cleanData(this.userItem.ID)
                successText = `${this.$t('user.resetUser1')}${this.userItem.Username}${this.$t('user.success')}`
            }
            if (result.Msg === 'success') {
                this.$message({
                    message: successText,
                    type: 'success'
                })
            } else {
                this.$message.error(result.Msg)
            }
            this.refresh()
        },
        async handleUpdateUser() {
            if (this.userInfo.username === '' || this.userInfo.password === '') {
                this.$message.error(this.$t('inputNotNull'))
                return
            }
            if (this.userInfo.username === 'admin') {
                this.$message.error(this.$t('user.limitAdmin'))
                return
            }
            const formData = new FormData()
            formData.set('id', this.userItem.ID)
            formData.set('username', this.userInfo.username)
            formData.set('password', btoa(this.userInfo.password))
            const result = await updateUser(formData)
            if (result.Msg === 'success') {
                this.$message({
                    message: `${this.$t('user.modifyUser2')}${this.userInfo.username}${this.$t('user.success')}`,
                    type: 'success'
                })
                this.userInfo.username = ''
                this.userInfo.password = ''
            } else {
                this.$message.error(result.Msg)
            }
            this.userVisible = false
            this.refresh()
        },
        async handleAddUser() {
            if (this.userInfo.username === '' || this.userInfo.password === '') {
                this.$message.error(this.$t('inputNotNull'))
                return
            }
            if (this.userInfo.username === 'admin') {
                this.$message.error(this.$t('user.limitAdmin'))
                return
            }
            const formData = new FormData()
            formData.set('username', this.userInfo.username)
            formData.set('password', btoa(this.userInfo.password))
            const result = await addUser(formData)
            if (result.Msg === 'success') {
                this.$message({
                    message: `${this.$t('user.addUser2')}${this.userInfo.username}${this.$t('user.success')}`,
                    type: 'success'
                })
                this.userInfo.username = ''
                this.userInfo.password = ''
            } else {
                this.$message.error(result.Msg)
            }
            this.userVisible = false
            this.refresh()
        },
        refresh() {
            this.dataList = []
            this.domain = ''
            this.getUserList()
        },
        async getUserList() {
            const result = await userList()
            if (result.Msg === 'success') {
                this.dataList = result.Data.userList
                if (result.Data.domain !== '') {
                    this.domain = result.Data.domain
                } else {
                    const hostname = window.location.hostname
                    if (!isValidIP(hostname)) {
                        this.domain = hostname
                        const formData = new FormData()
                        formData.set('domain', this.domain)
                        this.$store.commit('SET_NPROGRESS', false)
                        this.$store.commit('SET_NOERROR', true)
                        try {
                            await setDomain(formData)
                        } catch (e) {
                            this.$store.commit('SET_NPROGRESS', true)
                            this.$store.commit('SET_NOERROR', false)
                        }
                    }
                }
            } else {
                this.$message.error(result.Msg)
            }
        }
    }
}
</script>

<style lang="scss">
.qrcodeCenter {
    margin: 0 auto;
    width: 200px;
}
</style>
