# AudioSystem

AudioSystem提供了系统层对`af`与`ap`的访问，其中最常用的函数便是：

```c++
sp<IAudioFlinger> get_audio_flinger(); // 获得audio flinger 指针
sp<IAudioPolicyService> get_audio_policy_service();  //获得audio policy service 指针
```

其数据结构如下，主要大致可以分为`af`相关的功能函数和`ap`相关的功能函数：

```c++
class AudioSystem
{
public:

    static status_t muteMicrophone(bool state);
    static status_t isMicrophoneMuted(bool *state);
    static status_t setMasterVolume(float value);
    static status_t getMasterVolume(float* volume);
    static status_t setMasterMute(bool mute);
    static status_t getMasterMute(bool* mute);
    static status_t setStreamVolume(audio_stream_type_t stream, float value,
                                    audio_io_handle_t output);
    static status_t getStreamVolume(audio_stream_type_t stream, float* volume,
                                    audio_io_handle_t output);
    static status_t setStreamMute(audio_stream_type_t stream, bool mute);
    static status_t getStreamMute(audio_stream_type_t stream, bool* mute);
    static status_t setMode(audio_mode_t mode);
    static status_t isStreamActive(audio_stream_type_t stream, bool *state, uint32_t inPastMs);
    static status_t isStreamActiveRemotely(audio_stream_type_t stream, bool *state,
            uint32_t inPastMs);
    static status_t isSourceActive(audio_source_t source, bool *state);
    static status_t setParameters(audio_io_handle_t ioHandle, const String8& keyValuePairs);
    static String8  getParameters(audio_io_handle_t ioHandle, const String8& keys);
    static status_t setParameters(const String8& keyValuePairs);
    static String8  getParameters(const String8& keys);
    static void setErrorCallback(audio_error_callback cb);
    static void setDynPolicyCallback(dynamic_policy_callback cb);
    static void setRecordConfigCallback(record_config_callback);
    static const sp<IAudioFlinger> get_audio_flinger();
    static float linearToLog(int volume);
    static int logToLinear(float volume);
    static size_t calculateMinFrameCount(
            uint32_t afLatencyMs, uint32_t afFrameCount, uint32_t afSampleRate,
            uint32_t sampleRate, float speed /*, uint32_t notificationsPerBufferReq*/);

    static status_t getOutputSamplingRate(uint32_t* samplingRate,
            audio_stream_type_t stream);
    static status_t getOutputFrameCount(size_t* frameCount,
            audio_stream_type_t stream);
    static status_t getOutputLatency(uint32_t* latency,
            audio_stream_type_t stream);
    static status_t getSamplingRate(audio_io_handle_t ioHandle,
                                          uint32_t* samplingRate);
    static status_t getFrameCount(audio_io_handle_t ioHandle,
                                  size_t* frameCount);
    static status_t getLatency(audio_io_handle_t output,
                               uint32_t* latency);
    static status_t getInputBufferSize(uint32_t sampleRate, audio_format_t format,
        audio_channel_mask_t channelMask, size_t* buffSize);

    static status_t setVoiceVolume(float volume);


    static status_t getRenderPosition(audio_io_handle_t output,
                                      uint32_t *halFrames,
                                      uint32_t *dspFrames);

    static uint32_t getInputFramesLost(audio_io_handle_t ioHandle);

    static audio_unique_id_t newAudioUniqueId(audio_unique_id_use_t use);

    static void acquireAudioSessionId(audio_session_t audioSession, pid_t pid);
    static void releaseAudioSessionId(audio_session_t audioSession, pid_t pid);

    static audio_hw_sync_t getAudioHwSyncForSession(audio_session_t sessionId);

    static status_t systemReady();

    static status_t getFrameCountHAL(audio_io_handle_t ioHandle,
                                     size_t* frameCount);
    enum sync_event_t {
        SYNC_EVENT_SAME = -1,           
        SYNC_EVENT_NONE = 0,
        SYNC_EVENT_PRESENTATION_COMPLETE,
        SYNC_EVENT_CNT,
    };

    static const uint32_t kSyncRecordStartTimeOutMs = 30000;

    static status_t setDeviceConnectionState(audio_devices_t device, audio_policy_dev_state_t state,
                                             const char *device_address, const char *device_name);
    static audio_policy_dev_state_t getDeviceConnectionState(audio_devices_t device,
                                                                const char *device_address);
    static status_t handleDeviceConfigChange(audio_devices_t device,
                                             const char *device_address,
                                             const char *device_name);
    static status_t setPhoneState(audio_mode_t state);
    static status_t setForceUse(audio_policy_force_use_t usage, audio_policy_forced_cfg_t config);
    static audio_policy_forced_cfg_t getForceUse(audio_policy_force_use_t usage);

    static status_t getOutputForAttr(const audio_attributes_t *attr,
                                     audio_io_handle_t *output,
                                     audio_session_t session,
                                     audio_stream_type_t *stream,
                                     pid_t pid,
                                     uid_t uid,
                                     const audio_config_t *config,
                                     audio_output_flags_t flags,
                                     audio_port_handle_t *selectedDeviceId,
                                     audio_port_handle_t *portId);
    static status_t startOutput(audio_io_handle_t output,
                                audio_stream_type_t stream,
                                audio_session_t session);
    static status_t stopOutput(audio_io_handle_t output,
                               audio_stream_type_t stream,
                               audio_session_t session);
    static void releaseOutput(audio_io_handle_t output,
                              audio_stream_type_t stream,
                              audio_session_t session);

    // Client must successfully hand off the handle reference to AudioFlinger via createRecord(),
    // or release it with releaseInput().
    static status_t getInputForAttr(const audio_attributes_t *attr,
                                    audio_io_handle_t *input,
                                    audio_session_t session,
                                    pid_t pid,
                                    uid_t uid,
                                    const String16& opPackageName,
                                    const audio_config_base_t *config,
                                    audio_input_flags_t flags,
                                    audio_port_handle_t *selectedDeviceId,
                                    audio_port_handle_t *portId);

    static status_t startInput(audio_port_handle_t portId,
                               bool *silenced);
    static status_t stopInput(audio_port_handle_t portId);
    static void releaseInput(audio_port_handle_t portId);
    static status_t initStreamVolume(audio_stream_type_t stream,
                                      int indexMin,
                                      int indexMax);
    static status_t setStreamVolumeIndex(audio_stream_type_t stream,
                                         int index,
                                         audio_devices_t device);
    static status_t getStreamVolumeIndex(audio_stream_type_t stream,
                                         int *index,
                                         audio_devices_t device);

    static uint32_t getStrategyForStream(audio_stream_type_t stream);
    static audio_devices_t getDevicesForStream(audio_stream_type_t stream);

    static audio_io_handle_t getOutputForEffect(const effect_descriptor_t *desc);
    static status_t registerEffect(const effect_descriptor_t *desc,
                                    audio_io_handle_t io,
                                    uint32_t strategy,
                                    audio_session_t session,
                                    int id);
    static status_t unregisterEffect(int id);
    static status_t setEffectEnabled(int id, bool enabled);
    static void clearAudioConfigCache();
    
    //=============================================================
    //audio policy service
    static const sp<IAudioPolicyService> get_audio_policy_service();
    static uint32_t getPrimaryOutputSamplingRate();
    static size_t getPrimaryOutputFrameCount();
    static status_t setLowRamDevice(bool isLowRamDevice, int64_t totalMemory);
    static bool isOffloadSupported(const audio_offload_info_t& info);
    static status_t checkAudioFlinger();
    static status_t listAudioPorts(audio_port_role_t role,
                                   audio_port_type_t type,
                                   unsigned int *num_ports,
                                   struct audio_port *ports,
                                   unsigned int *generation);

    static status_t getAudioPort(struct audio_port *port);

    static status_t createAudioPatch(const struct audio_patch *patch,
                                       audio_patch_handle_t *handle);

    static status_t releaseAudioPatch(audio_patch_handle_t handle);

    static status_t listAudioPatches(unsigned int *num_patches,
                                      struct audio_patch *patches,
                                      unsigned int *generation);
    static status_t setAudioPortConfig(const struct audio_port_config *config);


    static status_t acquireSoundTriggerSession(audio_session_t *session,
                                           audio_io_handle_t *ioHandle,
                                           audio_devices_t *device);
    static status_t releaseSoundTriggerSession(audio_session_t session);

    static audio_mode_t getPhoneState();

    static status_t registerPolicyMixes(const Vector<AudioMix>& mixes, bool registration);

    static status_t startAudioSource(const struct audio_port_config *source,
                                      const audio_attributes_t *attributes,
                                      audio_patch_handle_t *handle);
    static status_t stopAudioSource(audio_patch_handle_t handle);

    static status_t setMasterMono(bool mono);
    static status_t getMasterMono(bool *mono);

    static float    getStreamVolumeDB(
            audio_stream_type_t stream, int index, audio_devices_t device);

    static status_t getMicrophones(std::vector<media::MicrophoneInfo> *microphones);
    static status_t getSurroundFormats(unsigned int *numSurroundFormats,
                                       audio_format_t *surroundFormats,
                                       bool *surroundFormatsEnabled,
                                       bool reported);
    static status_t setSurroundFormatEnabled(audio_format_t audioFormat, bool enabled);

    class AudioPortCallback : public RefBase
    {
    public:

                AudioPortCallback() {}
        virtual ~AudioPortCallback() {}

        virtual void onAudioPortListUpdate() = 0;
        virtual void onAudioPatchListUpdate() = 0;
        virtual void onServiceDied() = 0;

    };

    static status_t addAudioPortCallback(const sp<AudioPortCallback>& callback);
    static status_t removeAudioPortCallback(const sp<AudioPortCallback>& callback);

    class AudioDeviceCallback : public RefBase
    {
    public:

                AudioDeviceCallback() {}
        virtual ~AudioDeviceCallback() {}

        virtual void onAudioDeviceUpdate(audio_io_handle_t audioIo,
                                         audio_port_handle_t deviceId) = 0;
    };

    static status_t addAudioDeviceCallback(const wp<AudioDeviceCallback>& callback,
                                           audio_io_handle_t audioIo);
    static status_t removeAudioDeviceCallback(const wp<AudioDeviceCallback>& callback,
                                              audio_io_handle_t audioIo);

    static audio_port_handle_t getDeviceIdForIo(audio_io_handle_t audioIo);

private:

    class AudioFlingerClient: public IBinder::DeathRecipient, public BnAudioFlingerClient
    {
    public:
        AudioFlingerClient() :
            mInBuffSize(0), mInSamplingRate(0),
            mInFormat(AUDIO_FORMAT_DEFAULT), mInChannelMask(AUDIO_CHANNEL_NONE) {
        }

        void clearIoCache();
        status_t getInputBufferSize(uint32_t sampleRate, audio_format_t format,
                                    audio_channel_mask_t channelMask, size_t* buffSize);
        sp<AudioIoDescriptor> getIoDescriptor(audio_io_handle_t ioHandle);

        virtual void binderDied(const wp<IBinder>& who);
        virtual void ioConfigChanged(audio_io_config_event event,
                                     const sp<AudioIoDescriptor>& ioDesc);


        status_t addAudioDeviceCallback(const wp<AudioDeviceCallback>& callback,
                                               audio_io_handle_t audioIo);
        status_t removeAudioDeviceCallback(const wp<AudioDeviceCallback>& callback,
                                           audio_io_handle_t audioIo);

        audio_port_handle_t getDeviceIdForIo(audio_io_handle_t audioIo);

    private:
        Mutex                               mLock;
        DefaultKeyedVector<audio_io_handle_t, sp<AudioIoDescriptor> >   mIoDescriptors;
        DefaultKeyedVector<audio_io_handle_t, Vector < wp<AudioDeviceCallback> > >
                                                                        mAudioDeviceCallbacks;
        // cached values for recording getInputBufferSize() queries
        size_t                              mInBuffSize;    // zero indicates cache is invalid
        uint32_t                            mInSamplingRate;
        audio_format_t                      mInFormat;
        audio_channel_mask_t                mInChannelMask;
        sp<AudioIoDescriptor> getIoDescriptor_l(audio_io_handle_t ioHandle);
    };

    class AudioPolicyServiceClient: public IBinder::DeathRecipient,
                                    public BnAudioPolicyServiceClient
    {
    public:
        AudioPolicyServiceClient() {
        }

        int addAudioPortCallback(const sp<AudioPortCallback>& callback);
        int removeAudioPortCallback(const sp<AudioPortCallback>& callback);
        bool isAudioPortCbEnabled() const { return (mAudioPortCallbacks.size() != 0); }

        // DeathRecipient
        virtual void binderDied(const wp<IBinder>& who);

        // IAudioPolicyServiceClient
        virtual void onAudioPortListUpdate();
        virtual void onAudioPatchListUpdate();
        virtual void onDynamicPolicyMixStateUpdate(String8 regId, int32_t state);
        virtual void onRecordingConfigurationUpdate(int event,
                        const record_client_info_t *clientInfo,
                        const audio_config_base_t *clientConfig,
                        const audio_config_base_t *deviceConfig, audio_patch_handle_t patchHandle);

    private:
        Mutex                               mLock;
        Vector <sp <AudioPortCallback> >    mAudioPortCallbacks;
    };

    static audio_io_handle_t getOutput(audio_stream_type_t stream);
    static const sp<AudioFlingerClient> getAudioFlingerClient();
    static sp<AudioIoDescriptor> getIoDescriptor(audio_io_handle_t ioHandle);

    static sp<AudioFlingerClient> gAudioFlingerClient;
    static sp<AudioPolicyServiceClient> gAudioPolicyServiceClient;
    friend class AudioFlingerClient;
    friend class AudioPolicyServiceClient;

    static Mutex gLock;      // protects gAudioFlinger and gAudioErrorCallback,
    static Mutex gLockAPS;   // protects gAudioPolicyService and gAudioPolicyServiceClient
    static sp<IAudioFlinger> gAudioFlinger;
    static audio_error_callback gAudioErrorCallback;
    static dynamic_policy_callback gDynPolicyCallback;
    static record_config_callback gRecordConfigCallback;

    static size_t gInBuffSize;
    // previous parameters for recording buffer size queries
    static uint32_t gPrevInSamplingRate;
    static audio_format_t gPrevInFormat;
    static audio_channel_mask_t gPrevInChannelMask;

    static sp<IAudioPolicyService> gAudioPolicyService;
};

};  // namespace android

#endif  /*ANDROID_AUDIOSYSTEM_H_*/

```



## audio flinger 相关操作函数

audio flinger相关的函数大多是通过先获得全局的audioflinger，在通过获得的audioflinger进行操作如：

```c++
status_t AudioSystem::muteMicrophone(bool state)
{
    const sp<IAudioFlinger>& af = AudioSystem::get_audio_flinger();
    if (af == 0) return PERMISSION_DENIED;
    return af->setMicMute(state);
}
```

而AF最终通过HAL层，将上述的函数实现，我们在此处主要关注一下audio flinger函数主要的作用对象：

- `audio_stream_type_t`

- `audio_io_handle_t`

- `audio_source_t`

- `audio_unique_id_use_t`

- `audio_session_t`

### audio_stream_type_t



### audio_io_handle_t
### audio_source_t
### audio_unique_id_ust_t
### audio_session_t




## audio policy service相关操作函数