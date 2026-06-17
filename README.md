#include "EGL/egl.h"
#include "GLES3/gl3.h"
#include <android/input.h>
#include <android/native_window.h>
#include <dlfcn.h>
#include <time.h>
#include <unistd.h>
#include <chrono>
#include <cctype>
#include <cstdlib>
#include <unordered_map>
#include <vector>
#include <fstream>
#include <string>
#include "nlohmann/json.hpp"
#include "ImGui/imgui.h"
#include "ImGui/backends/imgui_impl_android.h"
#include "ImGui/backends/imgui_impl_opengl3.h"
#include "ImGui/misc/fonts/Roboto-Medium.h"
#include "ImGui/misc/fonts/FontAwesome-Brands.h"
#include "ImGui/misc/fonts/FontAwesome-Solid.h"
#include "Includes/Logger.h"
#include "Includes/obfuscate.h"
#include "Includes/Utils.h"
#include "KittyMemory/KittyMemory.h"
#include "And64InlineHook/And64InlineHook.hpp"
#include "Substrate/SubstrateHook.h"
#include "Substrate/CydiaSubstrate.h"
#include "Tools/Call_Tools.h"
#include "Tools/MonoString.h"
#include "Il2cpp/il2cpp-class.h"
#include "Il2cpp/Il2cpp.h"

// il2cpp_runtime_invoke — resolved by Il2cpp::Init() at runtime
extern Il2CppObject *(*il2cpp_runtime_invoke)(MethodInfo *method, void *obj, void **params, Il2CppException **exc);
extern void *(*il2cpp_unity_liveness_allocate_struct)(Il2CppClass *filter, int max_object_count, il2cpp_register_object_callback callback, void *userdata, il2cpp_liveness_reallocate_callback reallocate);
extern void (*il2cpp_unity_liveness_calculation_from_statics)(void *state);
extern void (*il2cpp_unity_liveness_finalize)(void *state);
extern void (*il2cpp_unity_liveness_free_struct)(void *state);
extern void (*il2cpp_gchandle_foreach_get_target)(void (*func)(void *data, void *userData), void *userData);
extern void (*il2cpp_unity_liveness_calculation_from_root)(Il2CppObject *root, void *state);
extern void (*il2cpp_gc_foreach_heap)(void (*func)(void *data, void *userData), void *userData);
extern Il2CppClass *(*il2cpp_object_get_class)(Il2CppObject *obj);
extern bool (*il2cpp_class_is_assignable_from)(Il2CppClass *klass, Il2CppClass *oklass);
extern const char *(*il2cpp_class_get_name)(Il2CppClass *klass);
extern void *(*il2cpp_object_unbox)(Il2CppObject *obj);
static bool IsBadReadPtr(void *p);
static Il2CppClass* SafeGetClass(void *obj);
extern Il2CppObject *(*il2cpp_object_new)(Il2CppClass *klass);
extern void (*il2cpp_field_set_value_object)(Il2CppObject *instance, FieldInfo *field, Il2CppObject *value);
extern void (*il2cpp_runtime_object_init)(Il2CppObject *obj);
extern _Il2CppArray *(*il2cpp_array_new_specific)(Il2CppClass *arrayTypeInfo, il2cpp_array_size_t length);
extern Il2CppClass *(*il2cpp_class_get_element_class)(Il2CppClass *klass);
extern int (*il2cpp_array_element_size)(Il2CppClass *array_class);
#include "Includes/Macros.h"
#include "Hook.h"
#include "Includes/dex_data.h"
#include "Includes/JNIStuff.h"
#include "LuaConsole.h"

void StartVulkanHooks();
bool IsVulkanOverlayHooked();
bool IsOverlayMenuVisible();
bool IsOverlaySurfaceReady();
bool IsGameRenderFallbackActive();
void SetGameRenderFallbackActive(bool active);
void SetOverlayMenuVisible(bool visible);
void UpdateOverlayMenuBounds(float x, float y, float w, float h);
void UpdateOverlayLogoBounds(float x, float y, float w, float h);
void UpdateOverlayFloatWinBounds(float x, float y, float w, float h);

struct FloatWinRect
{
  bool active = false;
  float x = 0.0f, y = 0.0f, w = 0.0f, h = 0.0f;
};
static FloatWinRect s_redirectRect;
static FloatWinRect s_instancesRect;
static FloatWinRect s_filterRect;
static FloatWinRect s_callerPickerRect;
bool EnsureOverlayLogoTexture();
ImTextureID GetOverlayLogoTexture();
static void SaveActiveSearchTabState();
static void LoadSearchTabState(int index);
void SaveConfig();
void LoadConfig();

static volatile bool g_Il2cppReady = false;

struct PathNode {
    std::string className;
    std::string label;
};

struct QueuedSetField {
    void *tabRootId;
    std::string className;
    std::string fieldName;
    std::string setterName;
    std::string valueExpr;
    std::vector<PathNode> historyPath;
};

static std::vector<QueuedSetField> g_BatchFieldQueue;
static bool g_BatchScriptMode = false;

static std::string GetPathKey(const QueuedSetField &q)
{
    std::string key;
    if (q.historyPath.size() > 1)
    {
        key += q.historyPath[0].className;
        for (size_t i = 1; i < q.historyPath.size(); ++i)
        {
            key += "||" + q.historyPath[i].label;
        }
    }
    else
    {
        key += q.className;
    }
    return key;
}

static constexpr const char *ICON_FA_YOUTUBE = "\xef\x85\xa7";
static constexpr const char *ICON_FA_TELEGRAM = "\xef\x8b\x86";
static constexpr const char *ICON_FA_FACEBOOK = "\xef\x82\x9a";
static constexpr const char *ICON_FA_GLOBE = "\xef\x82\xac";
static constexpr const char *ICON_FA_CARET_DOWN = "\xef\x83\x97";

static void OpenAndroidUrl(const char *url)
{
  if (!url || !jvm)
    return;

  bool attached = false;
  JNIEnv *env = GetEnv(&attached);
  if (!env)
    return;

  jobject activity = GetCurrentActivity(env);
  if (!activity)
  {
    if (attached)
      jvm->DetachCurrentThread();
    return;
  }

  jclass uriClass = env->FindClass("android/net/Uri");
  jclass intentClass = env->FindClass("android/content/Intent");
  jclass activityClass = env->GetObjectClass(activity);
  if (uriClass && intentClass && activityClass)
  {
    jmethodID parseMethod = env->GetStaticMethodID(uriClass, "parse", "(Ljava/lang/String;)Landroid/net/Uri;");
    jmethodID intentCtor = env->GetMethodID(intentClass, "<init>", "(Ljava/lang/String;Landroid/net/Uri;)V");
    jmethodID startActivityMethod = env->GetMethodID(activityClass, "startActivity", "(Landroid/content/Intent;)V");
    jstring actionView = env->NewStringUTF("android.intent.action.VIEW");
    jstring jurl = env->NewStringUTF(url);
    jobject uri = parseMethod ? env->CallStaticObjectMethod(uriClass, parseMethod, jurl) : nullptr;
    jobject intent = (intentCtor && uri) ? env->NewObject(intentClass, intentCtor, actionView, uri) : nullptr;
    if (intent && startActivityMethod)
      env->CallVoidMethod(activity, startActivityMethod, intent);

    if (env->ExceptionCheck())
      env->ExceptionClear();
    if (intent)
      env->DeleteLocalRef(intent);
    if (uri)
      env->DeleteLocalRef(uri);
    env->DeleteLocalRef(jurl);
    env->DeleteLocalRef(actionView);
  }

  if (activityClass)
    env->DeleteLocalRef(activityClass);
  if (intentClass)
    env->DeleteLocalRef(intentClass);
  if (uriClass)
    env->DeleteLocalRef(uriClass);
  env->DeleteLocalRef(activity);
  if (attached)
    jvm->DetachCurrentThread();
}

static void SetupMenuFonts(ImGuiIO &io, float baseSize)
{
  ImFontConfig font_cfg;
  font_cfg.OversampleH = 3;
  font_cfg.OversampleV = 2;
  font_cfg.PixelSnapH = true;
  font_cfg.RasterizerMultiply = 1.08f;
  font_cfg.FontDataOwnedByAtlas = false;

  io.Fonts->Clear();
  io.FontDefault = io.Fonts->AddFontFromMemoryTTF(Roboto_Medium,
                                                  sizeof(Roboto_Medium),
                                                  baseSize,
                                                  &font_cfg,
                                                  io.Fonts->GetGlyphRangesCyrillic());

  ImFontConfig icon_cfg;
  icon_cfg.MergeMode = true;
  icon_cfg.PixelSnapH = true;
  icon_cfg.FontDataOwnedByAtlas = false;
  icon_cfg.GlyphMinAdvanceX = baseSize;

  static const ImWchar brand_ranges[] = {0xf09a, 0xf09a, 0xf167, 0xf167, 0xf2c6, 0xf2c6, 0};
  static const ImWchar solid_ranges[] = {0xf0ac, 0xf0ac, 0xf0d7, 0xf0d7, 0};

  io.Fonts->AddFontFromMemoryTTF(FontAwesomeBrands,
                                 FontAwesomeBrandsLen,
                                 baseSize,
                                 &icon_cfg,
                                 brand_ranges);
  io.Fonts->AddFontFromMemoryTTF(FontAwesomeSolid,
                                 FontAwesomeSolidLen,
                                 baseSize,
                                 &icon_cfg,
                                 solid_ranges);
}

__attribute__((constructor)) static void OnLibraryLoaded()
{
  LOGI("libSilverv7 loaded into process");
}

static bool HookAllLevel = false;
static int SymbolLevelValue = 100;
static int SkillTreeLevelValue = 150;
static int PvpLevelValue = 150;
static bool g_AllInOneEnabled = false;
static bool g_PetLevelEnabled = false;
static bool g_StatEnabled = false;
static bool g_UnlockQuestEnabled = false;
static bool g_OneHitEnabled = false;
static bool g_SpeedEnabled = false;
static float g_TimeScaleValue = 1.0f;
static int g_RubyCount = 1;
static bool g_ShowSpeedControl = false;
static clock_t mainThread_LastTickLogTime = 0;
static float g_FontScale = 1.0f;
static ImGuiStyle g_BaseMenuStyle;
static bool g_BaseMenuStyleCaptured = false;
static bool g_KeyUnlocked = false;
static char g_KeyInput[64] = {};
static char g_SavedKey[64] = "";
static bool g_KeyInvalid = false;
static const char *g_MasterKey = "Silverv7";
static bool g_ConfigLoaded = false;
static std::mutex g_Il2cppMutex;
static int g_ActiveTabIndex = 0;
static bool g_RequestToolsTab = false;
bool g_AutoSaveScript = false;
static std::string g_LastSelectedClassFullName;
static std::string g_LastSelectedMethodName;
static bool g_RestoreSelectionPending = true;
static bool g_FitInstancesWindowNextOpen = false;
static bool g_FitCallerObjectPickerNextOpen = false;
static bool g_ToolsInspectorFullView = false;
static float g_MenuPosSavedX = -1.0f;
static float g_MenuPosSavedY = -1.0f;
static float g_MenuSizeSavedW = -1.0f;
static float g_MenuSizeSavedH = -1.0f;
static bool g_MenuCollapsedSaved = false;
static bool g_SetWindowSettingsFromConfig = false;
static float g_SaveConfigDirtyTimer = 0.0f;
static bool g_ShowMenuToggler = true;
static bool g_SilentMode = false;
static bool g_ShowTabScript = true;
static bool g_ShowTabTools = true;
static bool g_ShowTabPatched = false;
static bool g_ShowTabDump = false;
static bool g_ShowTabTracer = false;

static void CaptureBaseMenuStyle()
{
  g_BaseMenuStyle = ImGui::GetStyle();
  g_BaseMenuStyleCaptured = true;
}

static void ApplyMenuFontScale(float scale)
{
  if (!g_BaseMenuStyleCaptured)
    CaptureBaseMenuStyle();

  float oldScale = g_FontScale;
  g_FontScale = scale;
  ImGui::GetIO().FontGlobalScale = g_FontScale;
  ImGui::GetStyle() = g_BaseMenuStyle;
  ImGui::GetStyle().ScaleAllSizes(g_FontScale);

  if (oldScale > 0.0f && oldScale != scale)
  {
    float ratio = scale / oldScale;
    if (g_MenuSizeSavedW > 0.0f)
      g_MenuSizeSavedW *= ratio;
    if (g_MenuSizeSavedH > 0.0f)
      g_MenuSizeSavedH *= ratio;
    g_SetWindowSettingsFromConfig = true;
  }
}

struct MethodParamEntry
{
  std::string name;
  std::string type;
};

struct MethodEntry
{
  std::string name;
  std::string signature;
  std::string returnType;
  uint64_t rva;
  int paramCount;
  std::vector<MethodParamEntry> params;
  void *methodPointer = nullptr;
};

struct FieldEntry
{
  std::string name;
  std::string type;
  size_t offset;
  std::string signature;
  bool isStatic;
};

struct ClassEntry
{
  std::string name;
  std::string ns;
  bool isEnum;
  bool isStruct;
  std::string parentName;
  std::vector<MethodEntry> methods;
  std::vector<FieldEntry> fields;
};

struct ImageEntry
{
  std::string name;
  std::vector<ClassEntry> classes;
};

struct FlatResultEntry
{
  std::string name;
  std::string ns;
  bool isEnum;
  bool isStruct;
  std::string parentName;
  std::vector<MethodEntry> methods;
  std::vector<FieldEntry> fields;
};

#include <mutex>
#include <algorithm>
#include <chrono>

#ifndef UNTAG_PTR
#define UNTAG_PTR(ptr) ((__typeof__(ptr))((uintptr_t)(ptr) & 0x00FFFFFFFFFFFFFFULL))
#endif

struct TraceState
{
  void *methodPointer;
  uint64_t rva;
  std::string className;
  std::string methodName;
  std::string fullName;
  int hitCount;
  bool enabled;
  std::chrono::steady_clock::time_point lastTime;
  int countInPeriod;
};

struct TraceLogEntry
{
  void *methodPointer;
  std::string className;
  std::string methodName;
  std::string fullName;
};

static std::mutex g_TraceMutex;
static std::unordered_map<void *, TraceState> g_TracedMethodsMap;
static std::vector<void *> g_TracedMethodsOrder;
static std::vector<TraceLogEntry> g_TraceLogs;

// Global settings
static int g_TraceMaxSpam = 100;
static bool g_TraceLimitEnabled = true;
static bool g_TraceInstantRemove = false;
static bool g_TraceDisplayClassName = true;
static bool g_TraceDisplayAddress = false;
static bool g_TraceSortByHit = false;
static int g_TraceMode = 0; // 0: Traced Methods, 1: Trace Logs

static bool IsAnyMethodTraced(const FlatResultEntry &selectedItem)
{
  std::lock_guard<std::mutex> lock(g_TraceMutex);
  for (auto &method : selectedItem.methods)
  {
    if (!method.methodPointer)
      continue;
    void *ptr = UNTAG_PTR(method.methodPointer);
    auto it = g_TracedMethodsMap.find(ptr);
    if (it != g_TracedMethodsMap.end() && it->second.enabled)
    {
      return true;
    }
  }
  return false;
}

static std::vector<ImageEntry> g_MetadataCache;
static std::vector<FlatResultEntry> g_FilteredList;
static bool g_MetadataLoaded = false;
#define FIELD_ATTRIBUTE_STATIC 0x0010
#define FIELD_ATTRIBUTE_LITERAL 0x0040
static bool g_FilterNeedsUpdate = true;
static int g_TotalImages = 0;
static int g_TotalClasses = 0;
static int g_TotalMethods = 0;
static int g_TotalFields = 0;

static int g_SelectedImageIdx = 0;
static int g_SelectedFlatIdx = -1;

static char g_PatternSearch[128] = "";
static char g_LastPatternSearch[128] = "";
static char g_FilterSearch[128] = "";
static char g_LastFilterSearch[128] = "";

static bool g_SearchAllImages = false;
static bool g_FilterCaseSensitive = false;
static bool g_FilterStartsWith = false;
static bool g_FilterEndsWith = false;
static bool g_FilterSimpleMode = true;
static int g_FilterType = 0;
static int g_FilterMethodOption = 1;
static bool g_SplitLayout = false;
static bool g_InspectorFiltered = false;
static bool g_ShowFilterOptionsWindow = false;
static bool g_FitFilterOptionsWindowNextOpen = false;
static ImVec2 g_FilterOptionsWindowPos = ImVec2(0.0f, 0.0f);
static bool g_ShowRedirectChooserWindow = false;
static bool g_FitRedirectChooserWindowNextOpen = false;
static ImVec2 g_RedirectChooserWindowPos = ImVec2(0.0f, 0.0f);
static std::string g_RedirectChooserId;
static std::string g_RedirectSourceClass;
static MethodEntry g_RedirectSourceMethod;

struct RawInspectTab
{
  void *data;
  Il2CppClass *klass;
  std::string label;
};

struct InspectHistoryNode
{
  enum Type
  {
    TYPE_OBJECT,
    TYPE_RAW
  };
  Type type;
  void *ptr; // obj or data
  Il2CppClass *klass;
  std::string label;
};

struct SearchTabState
{
  enum TabType
  {
    TYPE_SEARCH,
    TYPE_INSPECT_OBJECT,
    TYPE_INSPECT_RAW
  };
  TabType type = TYPE_SEARCH;

  std::string title;
  int selectedImageIdx = 0;
  bool searchAllImages = false;
  std::string filter;
  int filterType = 0;
  int filterMethodOption = 1;
  bool splitLayout = false;
  int selectedFlatIdx = -1;
  std::string selectedMethod;
  std::vector<FlatResultEntry> results;

  // Search tab selection state
  std::string lastSelectedClassFullName;

  // Inspect tab fields
  void *inspectPtr = nullptr;
  Il2CppClass *inspectKlass = nullptr;
  std::string inspectLabel;
};
static std::vector<SearchTabState> g_SearchTabs;
static int g_ActiveSearchTab = 0;
static bool g_RequestSearchTabSelection = true;
static int g_NextSearchTabId = 1;

static std::vector<uint32_t> g_ScannedGCHandles;
static char g_FindObjectsFilter[128] = "";
static void *g_InspectedObject = nullptr;
static std::vector<uint32_t> g_InspectedObjectTabs;
static bool g_SelectInspectedObjectTab = false;
static std::vector<RawInspectTab> g_RawInspectTabs;
static std::unordered_map<void *, std::vector<InspectHistoryNode>> g_TabHistories;
static std::unordered_map<void *, uint32_t> g_InspectedObjectHandles;

static void ClearScannedObjects()
{
  for (uint32_t handle : g_ScannedGCHandles)
  {
    if (handle)
    {
      Il2cpp::GCHandle_Free(handle);
    }
  }
  g_ScannedGCHandles.clear();
}

static void PinInspectedObject(void *obj)
{
  if (!obj) return;
  if (g_InspectedObjectHandles.find(obj) == g_InspectedObjectHandles.end())
  {
    uint32_t handle = Il2cpp::GCHandle_New((Il2CppObject *)obj, false);
    g_InspectedObjectHandles[obj] = handle;
  }
}

static void UnpinInspectedObject(void *obj)
{
  if (!obj) return;
  int count = 0;
  for (const auto &tab : g_SearchTabs)
  {
    if (tab.type == SearchTabState::TYPE_INSPECT_OBJECT && tab.inspectPtr == obj)
    {
      count++;
    }
  }
  if (count == 0)
  {
    auto it = g_InspectedObjectHandles.find(obj);
    if (it != g_InspectedObjectHandles.end())
    {
      if (it->second)
        Il2cpp::GCHandle_Free(it->second);
      g_InspectedObjectHandles.erase(it);
    }
  }
}

static void ClearInspectedObjects()
{
  for (auto &pair : g_InspectedObjectHandles)
  {
    if (pair.second)
      Il2cpp::GCHandle_Free(pair.second);
  }
  g_InspectedObjectHandles.clear();
}

static void RenderInspectBreadcrumbs(std::vector<InspectHistoryNode> &history)
{
  for (size_t i = 0; i < history.size(); ++i)
  {
    if (i > 0)
    {
      ImGui::SameLine(0, 0);
      if (history[i].label.rfind("[", 0) == 0)
      {
        ImGui::TextUnformatted(" ");
        ImGui::SameLine(0, 0);
      }
      else
      {
        ImGui::TextUnformatted(" . ");
        ImGui::SameLine(0, 0);
      }
    }

    std::string label = history[i].label;
    ImGui::PushID((int)i);

    bool isLast = (i == history.size() - 1);
    if (isLast)
    {
      ImGui::PushStyleColor(ImGuiCol_Text, ImVec4(0.95f, 0.95f, 0.95f, 1.00f));
      ImGui::TextUnformatted(label.c_str());
      ImGui::PopStyleColor();
    }
    else
    {
      ImGui::PushStyleColor(ImGuiCol_Text, ImVec4(0.65f, 0.65f, 0.70f, 1.00f));
      ImGui::TextUnformatted(label.c_str());
      if (ImGui::IsItemHovered())
      {
        ImGui::SetMouseCursor(ImGuiMouseCursor_Hand);
        ImVec2 min = ImGui::GetItemRectMin();
        ImVec2 max = ImGui::GetItemRectMax();
        min.y = max.y; // Vẽ đường gạch chân ở dưới text
        ImGui::GetWindowDrawList()->AddLine(min, max, ImGui::GetColorU32(ImGuiCol_Text));
      }
      ImGui::PopStyleColor();

      if (ImGui::IsItemClicked())
      {
        history.resize(i + 1);
        ImGui::PopID();
        break; // Thoát vòng lặp vì kích thước history đã thay đổi
      }
    }

    ImGui::PopID();
  }

  ImGui::Separator();
  ImGui::Spacing();
}
static int g_SelectRawInspectTab = -1;
static char g_ManualInputAddress[32] = "";
static char g_ObjectFieldFilter[128] = "";
static bool g_IncludeProperties = false;
static bool g_ScanRequested = false;
static std::string g_ScanClassName = "";
static bool g_ScanInProgress = false;
static bool g_ShowInstancesWindow = false;
static bool g_CallerObjectPickerOpen = false;
static std::string g_CallerObjectPickerClassName;
static std::string g_CallerObjectPickerTitle;
static std::string g_CallerObjectPickerSlotKey;
static std::unordered_map<std::string, uintptr_t> g_CallerObjectSelections;
static std::unordered_map<std::string, uint32_t> g_CallerGCHandles;

static void SetCallerObjectSelection(const std::string &slotKey, uintptr_t address)
{
  auto it = g_CallerGCHandles.find(slotKey);
  if (it != g_CallerGCHandles.end() && it->second != 0)
  {
    Il2cpp::GCHandle_Free(it->second);
    g_CallerGCHandles.erase(it);
  }

  if (address == 0)
  {
    g_CallerObjectSelections[slotKey] = 0;
  }
  else
  {
    g_CallerObjectSelections[slotKey] = address;
    uint32_t handle = Il2cpp::GCHandle_New((Il2CppObject *)address, false);
    if (handle != 0)
    {
      g_CallerGCHandles[slotKey] = handle;
    }
  }
}

void RegisterCreatedObject(void *obj, const char *slotKey)
{
  if (!obj)
    return;

  bool found = false;
  for (uint32_t handle : g_ScannedGCHandles)
  {
    if (Il2cpp::GCHandle_GetTarget(handle) == obj)
    {
      found = true;
      break;
    }
  }
  if (!found)
  {
    uint32_t handle = Il2cpp::GCHandle_New((Il2CppObject *)obj, false);
    g_ScannedGCHandles.insert(g_ScannedGCHandles.begin(), handle);
  }

  if (slotKey && slotKey[0] != '\0')
  {
    SetCallerObjectSelection(slotKey, (uintptr_t)obj);
  }
}
static std::unordered_map<std::string, std::vector<std::string>> g_CallerResultValues;
static std::unordered_map<std::string, std::string> g_CallerResultRaw;
static std::unordered_map<std::string, int> g_CallerCallCounts;
static std::string g_PendingCallerResultKey;
static std::string g_LastCallerCapturedOutput;

static void OpenInspectedObjectDump(void *obj)
{
  if (!obj)
    return;

  SaveActiveSearchTabState();

  int existingIdx = -1;
  for (int i = 0; i < (int)g_SearchTabs.size(); ++i)
  {
    if (g_SearchTabs[i].type == SearchTabState::TYPE_INSPECT_OBJECT && g_SearchTabs[i].inspectPtr == obj)
    {
      existingIdx = i;
      break;
    }
  }

  if (existingIdx != -1)
  {
    LoadSearchTabState(existingIdx);
  }
  else
  {
    SearchTabState tab;
    tab.type = SearchTabState::TYPE_INSPECT_OBJECT;
    tab.inspectPtr = obj;
    tab.title = "";
    g_SearchTabs.push_back(tab);
    PinInspectedObject(obj);
    LoadSearchTabState((int)g_SearchTabs.size() - 1);
  }

  g_RequestSearchTabSelection = true;
  g_RequestToolsTab = true;
  g_ShowInstancesWindow = false;
  UpdateOverlayFloatWinBounds(0.0f, 0.0f, 0.0f, 0.0f);
  SaveConfig();
}

static void OpenRawInspectTab(void *data, Il2CppClass *klass, const std::string &label)
{
  if (!data || !klass)
    return;

  SaveActiveSearchTabState();

  int existingIdx = -1;
  for (int i = 0; i < (int)g_SearchTabs.size(); ++i)
  {
    if (g_SearchTabs[i].type == SearchTabState::TYPE_INSPECT_RAW && g_SearchTabs[i].inspectPtr == data)
    {
      existingIdx = i;
      break;
    }
  }

  if (existingIdx != -1)
  {
    LoadSearchTabState(existingIdx);
  }
  else
  {
    SearchTabState tab;
    tab.type = SearchTabState::TYPE_INSPECT_RAW;
    tab.inspectPtr = data;
    tab.inspectKlass = klass;
    tab.inspectLabel = label;
    tab.title = "";
    g_SearchTabs.push_back(tab);
    LoadSearchTabState((int)g_SearchTabs.size() - 1);
  }

  g_RequestSearchTabSelection = true;
  g_RequestToolsTab = true;
  SaveConfig();
}

struct CustomMenuItem
{
  std::string name;
  std::string script;
  bool enabled;
};
static std::vector<CustomMenuItem> g_CustomMenuItems;
static char g_NewItemName[128] = "";
static bool g_ShowCreatePopup = false;
static char g_CreateErrorMsg[128] = "";
static int g_EditingIndex = -1;
static char g_EditNameBuf[128] = "";
static int g_DeleteConfirmIndex = -1;

#include <sys/stat.h>
#include <sys/types.h>

static std::string GetConfigPath()
{
  return GetAppFilesDir() + "/modmenu/config/script_state.json";
}

static bool EnsureDir(const std::string &path)
{
  struct stat info;
  if (stat(path.c_str(), &info) != 0)
  {
    if (mkdir(path.c_str(), 0777) != 0)
    {
      LOGE("EnsureDir: failed to create %s", path.c_str());
      return false;
    }
    return true;
  }
  return (info.st_mode & S_IFDIR) != 0;
}

static bool EnsureModMenuDirs()
{
  std::string base = GetAppFilesDir();
  if (base.empty())
    return false;

  std::string modmenu = base + "/modmenu";
  EnsureDir(modmenu);

  std::string scripts = modmenu + "/scripts";
  EnsureDir(scripts);

  std::string config = modmenu + "/config";
  EnsureDir(config);

  return true;
}

void SaveConfig()
{
  if (!g_ConfigLoaded)
  {
    LoadConfig();
  }
  g_SaveConfigDirtyTimer = 0.0f;
  EnsureModMenuDirs();
  SaveActiveSearchTabState();

  using json = nlohmann::json;
  json j;
  j["activated"] = true;
  j["fontScale"] = g_FontScale;
  j["activeTab"] = g_ActiveTabIndex;
  j["symbolLevel"] = SymbolLevelValue;
  j["skillLevel"] = SkillTreeLevelValue;
  j["pvpLevel"] = PvpLevelValue;
  j["autoSaveScript"] = g_AutoSaveScript;
  j["showSpeedControl"] = g_ShowSpeedControl;
  j["speedEnabled"] = g_SpeedEnabled;
  j["timeScaleValue"] = g_TimeScaleValue;
  j["savedKey"] = std::string(g_SavedKey);
  j["lastSelectedClass"] = g_LastSelectedClassFullName;
  j["lastSelectedMethod"] = g_LastSelectedMethodName;
  j["toolsInspectorFullView"] = g_ToolsInspectorFullView;
  j["menuPosX"] = g_MenuPosSavedX;
  j["menuPosY"] = g_MenuPosSavedY;
  j["menuSizeW"] = g_MenuSizeSavedW;
  j["menuSizeH"] = g_MenuSizeSavedH;
  j["menuCollapsed"] = g_MenuCollapsedSaved;
  j["showMenuToggler"] = g_ShowMenuToggler;
  j["showTabScript"] = g_ShowTabScript;
  j["showTabTools"] = g_ShowTabTools;
  j["showTabPatched"] = g_ShowTabPatched;
  j["showTabDump"] = g_ShowTabDump;
  j["showTabTracer"] = g_ShowTabTracer;

  // Save Search Tabs (only save TYPE_SEARCH tabs)
  int activeSearchTabIdx = 0;
  if (g_ActiveSearchTab >= 0 && g_ActiveSearchTab < (int)g_SearchTabs.size())
  {
    if (g_SearchTabs[g_ActiveSearchTab].type == SearchTabState::TYPE_SEARCH)
    {
      activeSearchTabIdx = g_ActiveSearchTab;
    }
    else
    {
      // Quét ngược tìm tab TYPE_SEARCH gần nhất
      bool found = false;
      for (int k = g_ActiveSearchTab - 1; k >= 0; --k)
      {
        if (g_SearchTabs[k].type == SearchTabState::TYPE_SEARCH)
        {
          activeSearchTabIdx = k;
          found = true;
          break;
        }
      }
      if (!found)
      {
        // Quét xuôi
        for (int k = g_ActiveSearchTab + 1; k < (int)g_SearchTabs.size(); ++k)
        {
          if (g_SearchTabs[k].type == SearchTabState::TYPE_SEARCH)
          {
            activeSearchTabIdx = k;
            found = true;
            break;
          }
        }
      }
    }
  }

  int savedActiveIdx = 0;
  int currentSavedIdx = 0;
  json searchTabsJson = json::array();
  for (int i = 0; i < (int)g_SearchTabs.size(); ++i)
  {
    const auto &tab = g_SearchTabs[i];
    if (tab.type != SearchTabState::TYPE_SEARCH)
      continue;
    if (i == activeSearchTabIdx)
      savedActiveIdx = currentSavedIdx;

    json jt;
    jt["title"] = tab.title;
    jt["selectedImageIdx"] = tab.selectedImageIdx;
    jt["searchAllImages"] = tab.searchAllImages;
    jt["filter"] = tab.filter;
    jt["filterType"] = tab.filterType;
    jt["filterMethodOption"] = tab.filterMethodOption;
    jt["splitLayout"] = tab.splitLayout;
    jt["selectedFlatIdx"] = tab.selectedFlatIdx;
    jt["selectedMethod"] = tab.selectedMethod;
    searchTabsJson.push_back(jt);
    currentSavedIdx++;
  }
  j["searchTabs"] = searchTabsJson;
  j["activeSearchTab"] = savedActiveIdx;
  j["nextSearchTabId"] = g_NextSearchTabId;

  // Save Lua script text
  const char *script = LuaConsole_GetScriptText();
  if (script && script[0])
    j["scriptText"] = std::string(script);
  else
    j["scriptText"] = "";

  // Save script file path (from LuaConsole)
  j["lastOpenedScript"] = LuaConsole_IsFileOpened() ? LuaConsole_GetActiveScriptName() : "";
  j["isFileOpened"] = LuaConsole_IsFileOpened();

  json items = json::array();
  for (auto &item : g_CustomMenuItems)
  {
    json ji;
    ji["name"] = item.name;
    ji["script"] = item.script;
    ji["enabled"] = false;
    items.push_back(ji);
  }
  j["customItems"] = items;
  std::string path = GetConfigPath();
  std::ofstream f(path);
  if (f.is_open())
  {
    f << j.dump(2);
    LOGI("Config: saved to %s", path.c_str());
  }
  else
  {
    LOGE("Config: failed to write %s", path.c_str());
  }
}

void LoadConfig()
{
  if (g_ConfigLoaded)
    return;
  g_ConfigLoaded = true;

  EnsureModMenuDirs();
  std::string path = GetConfigPath();
  std::ifstream f(path);
  if (!f.is_open())
  {
    LOGI("Config: no existing config at %s", path.c_str());
    return;
  }
  LOGI("Config: loading from %s", path.c_str());

  try
  {
    using json = nlohmann::json;
    json j;
    f >> j;

    if (j.contains("fontScale"))
    {
      ApplyMenuFontScale(j["fontScale"].get<float>());
    }

    g_ActiveTabIndex = j.value("activeTab", 0);

    SymbolLevelValue = j.value("symbolLevel", 100);
    SkillTreeLevelValue = j.value("skillLevel", 150);
    PvpLevelValue = j.value("pvpLevel", 150);
    g_AutoSaveScript = j.value("autoSaveScript", false);
    g_ShowSpeedControl = j.value("showSpeedControl", false);
    g_SpeedEnabled = j.value("speedEnabled", false);
    g_TimeScaleValue = j.value("timeScaleValue", 1.0f);

    std::string savedKey = j.value("savedKey", "");
    strncpy(g_SavedKey, savedKey.c_str(), sizeof(g_SavedKey));
    if (!savedKey.empty() && (savedKey == "V7" || savedKey == "cilent" || savedKey == "client"))
    {
      strncpy(g_KeyInput, savedKey.c_str(), sizeof(g_KeyInput));
      g_KeyUnlocked = true;
      g_KeyInvalid = false;
      g_SilentMode = (savedKey == "cilent" || savedKey == "client");
    }
    g_LastSelectedClassFullName = j.value("lastSelectedClass", "");
    g_LastSelectedMethodName = j.value("lastSelectedMethod", "");
    g_ToolsInspectorFullView = j.value("toolsInspectorFullView", false);

    if (j.contains("menuPosX"))
    {
      g_MenuPosSavedX = j["menuPosX"].get<float>();
      g_MenuPosSavedY = j["menuPosY"].get<float>();
      g_MenuSizeSavedW = j["menuSizeW"].get<float>();
      g_MenuSizeSavedH = j["menuSizeH"].get<float>();
      g_MenuCollapsedSaved = j.value("menuCollapsed", false);
      g_SetWindowSettingsFromConfig = true;
    }
    g_ShowMenuToggler = j.value("showMenuToggler", true);
    g_ShowTabScript = j.value("showTabScript", true);
    g_ShowTabTools = j.value("showTabTools", true);
    g_ShowTabPatched = j.value("showTabPatched", false);
    g_ShowTabDump = j.value("showTabDump", false);
    g_ShowTabTracer = j.value("showTabTracer", false);

    if (j.contains("searchTabs"))
    {
      ClearInspectedObjects();
      g_SearchTabs.clear();
      for (const auto &jt : j["searchTabs"])
      {
        SearchTabState tab;
        std::string titleVal = jt.value("title", "");
        if (titleVal == "Search")
          titleVal = "";
        tab.title = titleVal;
        tab.selectedImageIdx = jt.value("selectedImageIdx", 0);
        tab.searchAllImages = jt.value("searchAllImages", false);
        tab.filter = jt.value("filter", "");
        tab.filterType = jt.value("filterType", 0);
        tab.filterMethodOption = jt.value("filterMethodOption", 1);
        tab.splitLayout = jt.value("splitLayout", false);
        tab.selectedFlatIdx = jt.value("selectedFlatIdx", -1);
        tab.selectedMethod = jt.value("selectedMethod", "");
        g_SearchTabs.push_back(tab);
      }
      g_ActiveSearchTab = j.value("activeSearchTab", 0);
      g_NextSearchTabId = j.value("nextSearchTabId", 1);

      if (!g_SearchTabs.empty())
      {
        if (g_ActiveSearchTab < 0 || g_ActiveSearchTab >= (int)g_SearchTabs.size())
          g_ActiveSearchTab = 0;
        LoadSearchTabState(g_ActiveSearchTab);
      }
    }

    g_RestoreSelectionPending = !g_LastSelectedClassFullName.empty();

    std::string savedScript = j.value("scriptText", "");
    if (!savedScript.empty())
      LuaConsole_SetScriptText(savedScript.c_str());

    std::string lastOpened = j.value("lastOpenedScript", "");
    bool isFileOpened = j.value("isFileOpened", false);
    if (isFileOpened && !lastOpened.empty())
    {
      LuaConsole_SetActiveScriptName(lastOpened.c_str());
      LuaConsole_SetFileOpened(true);
      LuaConsole_LoadNamedScript(lastOpened);
    }

    if (j.contains("customItems"))
    {
      g_CustomMenuItems.clear();
      for (auto &ji : j["customItems"])
      {
        CustomMenuItem item;
        item.name = ji.value("name", "");
        item.script = ji.value("script", "");
        item.enabled = false;
        if (!item.name.empty() && !item.script.empty())
          g_CustomMenuItems.push_back(item);
      }
    }
  }
  catch (...)
  {
    LOGE("Config: corrupted, using defaults");
  }
  LOGI("Config: loaded ok, %d custom items", (int)g_CustomMenuItems.size());
}

// ============================================================

// ============================================================
// Shared helper: find System.Numerics.BigInteger.ctor(Int64)
// ============================================================
static MethodInfo *FindBigIntegerCtorInt64()
{
  Il2CppClass *biClass = Il2cpp::FindClass("System.Numerics.BigInteger");
  if (!biClass)
    return nullptr;
  void *iter = nullptr;
  MethodInfo *method;
  while ((method = Il2cpp::GetClassMethods(biClass, &iter)) != nullptr)
  {
    if (strcmp(Il2cpp::GetMethodName(method), ".ctor") != 0)
      continue;
    if (Il2cpp::GetMethodParamCount(method) != 1)
      continue;
    Il2CppType *paramType = Il2cpp::GetMethodParam(method, 0);
    Il2CppClass *paramClass = Il2cpp::GetClassFromType(paramType);
    if (paramClass && strcmp(Il2cpp::GetClassName(paramClass), "Int64") == 0)
      return method;
  }
  return nullptr;
}

// ============================================================
// Game scale: level 1 = 1048576 (0x100000 = 2^20)
// So value X needs X * 1048576 for the raw BigInteger
// ============================================================
static const int64_t LEVEL_SCALE = 1048576LL;

// ============================================================
// Helper: set a BigNumber field, syncing BOTH BigNumber + internal BigInteger
// bigNumberAddr = address of the BigNumber struct
// value = display level (e.g. 100), will be multiplied by LEVEL_SCALE
// ============================================================
static bool SetBigNumberRaw(void *bigNumberAddr, int64_t value)
{
  // Cache BigNumber.ctor(Int64)
  static MethodInfo *bnCtor = nullptr;
  if (!bnCtor)
  {
    Il2CppClass *bnClass = Il2cpp::FindClass("BigNumber");
    if (!bnClass)
    {
      LOGE("SetBigNumberRaw: BigNumber class not found");
      return false;
    }
    void *iter = nullptr;
    MethodInfo *m;
    while ((m = Il2cpp::GetClassMethods(bnClass, &iter)) != nullptr)
    {
      if (strcmp(Il2cpp::GetMethodName(m), ".ctor") != 0)
        continue;
      if (Il2cpp::GetMethodParamCount(m) != 1)
        continue;
      Il2CppType *pt = Il2cpp::GetMethodParam(m, 0);
      Il2CppClass *pc = Il2cpp::GetClassFromType(pt);
      if (pc && strcmp(Il2cpp::GetClassName(pc), "Int64") == 0)
      {
        bnCtor = m;
        break;
      }
    }
    if (!bnCtor || !bnCtor->methodPointer)
    {
      LOGE("SetBigNumberRaw: BigNumber.ctor(Int64) not found");
      return false;
    }
    LOGI("SetBigNumberRaw: BigNumber.ctor(Int64) methodPointer=%p", bnCtor->methodPointer);
  }

  // Cache BigInteger.ctor(Int64) for _raw_number sync
  static MethodInfo *biCtor = nullptr;
  if (!biCtor)
  {
    biCtor = FindBigIntegerCtorInt64();
    if (!biCtor || !biCtor->methodPointer)
    {
      LOGE("SetBigNumberRaw: BigInteger.ctor(Int64) not found");
      return false;
    }
    LOGI("SetBigNumberRaw: BigInteger.ctor(Int64) methodPointer=%p", biCtor->methodPointer);
  }

  int64_t scaled = value * LEVEL_SCALE;

  // 1. Set BigNumber with RAW value (hien thi)
  void *bnParams[] = {&value};
  Il2CppException *exc = nullptr;
  il2cpp_runtime_invoke(bnCtor, bigNumberAddr, bnParams, &exc);
  if (exc)
  {
    LOGE("SetBigNumberRaw: BigNumber.ctor threw exception");
    return false;
  }

  // 2. Set BigInteger._raw_number with SCALED value (that su)
  uint8_t *rawAddr = reinterpret_cast<uint8_t *>(bigNumberAddr) + 0x18;
  void *biParams[] = {&scaled};
  Il2CppException *exc2 = nullptr;
  il2cpp_runtime_invoke(biCtor, rawAddr, biParams, &exc2);
  if (exc2)
  {
    LOGW("SetBigNumberRaw: BigInteger.ctor warning (BigNumber already set)");
  }

  return true;
}

// ============================================================
// Level Skill: set SymbolManager._soulLevel via BigInteger
// ============================================================
// Symbol Level: set ALL SymbolData.level via BigInteger
// ============================================================
namespace SymbolLevel
{
  static void DoSymbolLevel()
  {
    Il2CppClass *smClass = Il2cpp::FindClass("DarkIdle2.SymbolManager");
    if (!smClass)
    {
      LOGE("SymbolLevel: SymbolManager class not found");
      return;
    }

    FieldInfo *instanceField = Il2cpp::GetClassField(smClass, "_instance");
    if (!instanceField)
    {
      LOGE("SymbolLevel: _instance field not found");
      return;
    }

    void *symbolManager = nullptr;
    Il2cpp::GetFieldStaticValue(instanceField, &symbolManager);
    if (!symbolManager)
    {
      LOGE("SymbolLevel: SymbolManager singleton is null");
      return;
    }

    FieldInfo *listField = Il2cpp::GetClassField(smClass, "_symbolDataList");
    if (!listField)
    {
      LOGE("SymbolLevel: _symbolDataList field not found");
      return;
    }

    Il2CppObject *listObj = Il2cpp::GetFieldValueObject((Il2CppObject *)symbolManager, listField);
    if (!listObj)
    {
      LOGE("SymbolLevel: _symbolDataList is null");
      return;
    }

    void **itemsPtr = reinterpret_cast<void **>(reinterpret_cast<uint8_t *>(listObj) + 0x10);
    int *sizePtr = reinterpret_cast<int *>(reinterpret_cast<uint8_t *>(listObj) + 0x18);
    _Il2CppArray *arr = reinterpret_cast<_Il2CppArray *>(*itemsPtr);
    int count = *sizePtr;
    if (!arr || count <= 0)
    {
      LOGE("SymbolLevel: _symbolDataList empty");
      return;
    }
    LOGI("SymbolLevel: found %d symbols", count);

    int64_t value = SymbolLevelValue;
    int modified = 0;

    for (int i = 0; i < count; i++)
    {
      void **elemPtr = reinterpret_cast<void **>(reinterpret_cast<uint8_t *>(arr) + 0x20 + i * 8);
      void *symbolData = *elemPtr;
      if (!symbolData)
        continue;

      // SymbolData.level BigNumber at offset 0x18
      void *levelAddr = reinterpret_cast<uint8_t *>(symbolData) + 0x18;
      if (SetBigNumberRaw(levelAddr, value))
        modified++;
    }

    LOGI("SymbolLevel: SUCCESS! Set %d/%d symbols to %lld", modified, count, value);
  }

  static void SetEnabled(bool enable)
  {
    if (enable)
      DoSymbolLevel();
    LOGI("Symbol Level: %s", enable ? "triggered" : "disabled");
  }

  static void InstallHook()
  {
    LOGI("SymbolLevel: ready (ALL SymbolData.level via BigInteger)");
  }
} // namespace SymbolLevel

// ============================================================
// PvP Level: PvpManager._pvpData.upgradeData.levelList
// ============================================================
namespace PvpLevel
{
  static void DoPvpLevel()
  {
    Il2CppClass *pmClass = Il2cpp::FindClass("PvpManager");
    if (!pmClass)
    {
      LOGE("PvpLevel: PvpManager class not found");
      return;
    }

    FieldInfo *instanceField = Il2cpp::GetClassField(pmClass, "_instance");
    if (!instanceField)
    {
      LOGE("PvpLevel: _instance field not found");
      return;
    }

    void *pvpManager = nullptr;
    Il2cpp::GetFieldStaticValue(instanceField, &pvpManager);
    if (!pvpManager)
    {
      LOGE("PvpLevel: PvpManager singleton is null");
      return;
    }

    // _pvpData (class ptr) at offset 0x20
    void **_pvpDataPtr = reinterpret_cast<void **>(reinterpret_cast<uint8_t *>(pvpManager) + 0x20);
    void *pvpSaveData = *_pvpDataPtr;
    if (!pvpSaveData)
    {
      LOGE("PvpLevel: _pvpData is null");
      return;
    }

    // upgradeData (class ptr) at offset 0xD0 within PvpSaveData
    void **upgradePtr = reinterpret_cast<void **>(reinterpret_cast<uint8_t *>(pvpSaveData) + 0xD0);
    void *upgradeData = *upgradePtr;
    if (!upgradeData)
    {
      LOGE("PvpLevel: upgradeData is null");
      return;
    }

    // levelList is a CLASS (List<BigNumber>), dereference first!
    void **listPtr = reinterpret_cast<void **>(reinterpret_cast<uint8_t *>(upgradeData) + 0x10);
    void *listObj = *listPtr;
    if (!listObj)
    {
      LOGE("PvpLevel: levelList is null");
      return;
    }

    // List internal: _items at +0x10, _size at +0x18
    void **itemsPtr = reinterpret_cast<void **>(reinterpret_cast<uint8_t *>(listObj) + 0x10);
    int *sizePtr = reinterpret_cast<int *>(reinterpret_cast<uint8_t *>(listObj) + 0x18);
    _Il2CppArray *arr = reinterpret_cast<_Il2CppArray *>(*itemsPtr);
    int count = *sizePtr;
    if (!arr || count <= 0)
    {
      LOGE("PvpLevel: levelList empty");
      return;
    }
    LOGI("PvpLevel: found %d pvp upgrades", count);

    // Get actual BigNumber struct size
    Il2CppClass *bnClass = Il2cpp::FindClass("BigNumber");
    int bnSize = bnClass ? Il2cpp::GetClassSize(bnClass) : 0x30;
    LOGI("PvpLevel: BigNumber size=%d, list count=%d", bnSize, count);
    int64_t value = PvpLevelValue;
    int modified = 0;

    for (int i = 0; i < count; i++)
    {
      void *bnAddr = reinterpret_cast<uint8_t *>(arr) + 0x20 + i * bnSize;
      if (SetBigNumberRaw(bnAddr, value))
        modified++;
    }

    LOGI("PvpLevel: SUCCESS! Set %d/%d pvp upgrades to %lld", modified, count, value);
  }

  static void SetEnabled(bool enable)
  {
    if (enable)
      DoPvpLevel();
    LOGI("PvP Level: %s", enable ? "triggered" : "disabled");
  }

  static void InstallHook()
  {
    LOGI("PvpLevel: ready (PvpManager->upgradeData->levelList)");
  }
} // namespace PvpLevel

// ============================================================
// Skill Tree Level: InGameScene.Instance.player.integratedSkillTreeStat.level
// ============================================================
namespace SkillTreeLevel
{
  static void DoSkillTreeLevel()
  {
    // 1. Get InGameScene singleton
    Il2CppClass *igsClass = Il2cpp::FindClass("InGameScene");
    if (!igsClass)
    {
      LOGE("SkillTreeLevel: InGameScene class not found");
      return;
    }

    FieldInfo *instanceField = Il2cpp::GetClassField(igsClass, "Instance");
    if (!instanceField)
    {
      LOGE("SkillTreeLevel: Instance field not found");
      return;
    }

    void *inGameScene = nullptr;
    Il2cpp::GetFieldStaticValue(instanceField, &inGameScene);
    if (!inGameScene)
    {
      LOGE("SkillTreeLevel: InGameScene.Instance is null");
      return;
    }
    LOGI("SkillTreeLevel: InGameScene=%p", inGameScene);

    // 2. Get Player at offset 0x28
    FieldInfo *playerField = Il2cpp::GetClassField(igsClass, "player");
    if (!playerField)
    {
      LOGE("SkillTreeLevel: player field not found");
      return;
    }

    Il2CppObject *player = Il2cpp::GetFieldValueObject((Il2CppObject *)inGameScene, playerField);
    if (!player)
    {
      LOGE("SkillTreeLevel: Player is null");
      return;
    }
    LOGI("SkillTreeLevel: Player=%p", player);

    // 3. integratedSkillTreeStat is a CLASS (pointer), dereference first!
    void **itsPtr = reinterpret_cast<void **>(reinterpret_cast<uint8_t *>(player) + 0x1C0);
    void *its = *itsPtr;
    if (!its)
    {
      LOGE("SkillTreeLevel: integratedSkillTreeStat is null");
      return;
    }
    LOGI("SkillTreeLevel: IntegratedSkillTreeStat=%p", its);

    // 4. level BigNumber at offset 0x10 within IntegratedSkillTreeStat
    void *levelAddr = reinterpret_cast<uint8_t *>(its) + 0x10;

    if (SetBigNumberRaw(levelAddr, SkillTreeLevelValue))
      LOGI("SkillTreeLevel: SUCCESS! integratedSkillTreeStat.level = %d", SkillTreeLevelValue);
    else
      LOGE("SkillTreeLevel: SetBigNumberRaw failed");
  }

  static void SetEnabled(bool enable)
  {
    if (enable)
      DoSkillTreeLevel();
    LOGI("Skill Tree Level: %s", enable ? "triggered" : "disabled");
  }

  static void InstallHook()
  {
    LOGI("SkillTreeLevel: ready (InGameScene->player->integratedSkillTreeStat.level)");
  }
} // namespace SkillTreeLevel

// ============================================================
// Extra Cheats: Character, Engrave, Bless, Artifact, Item
// ============================================================
namespace ExtraCheat
{
  struct ScanCtx
  {
    Il2CppClass *targetClass;
    int count;
  };

  static void ScanCallback(void *obj, void *userData)
  {
    ScanCtx *ctx = reinterpret_cast<ScanCtx *>(userData);
    if (SafeGetClass(obj) == ctx->targetClass)
      ctx->count++;
  }

  static void DoExtraCheat()
  {
    int total = 0;

    // --- 1. CharacterData.level = 1100 ---
    {
      Il2CppClass *klass = Il2cpp::FindClass("DarkIdle2.CharacterData");
      if (klass)
      {
        ScanCtx ctx = {klass, 0};
        il2cpp_gc_foreach_heap(ScanCallback, &ctx);
        LOGI("ExtraCheat: CharacterData scan found %d", ctx.count);
        total += ctx.count;
      }
    }

    // --- 2. EngraveData via AwakenManager._engraveData ---
    {
      Il2CppClass *amClass = Il2cpp::FindClass("DarkIdle2.AwakenManager");
      if (amClass)
      {
        FieldInfo *instF = Il2cpp::GetClassField(amClass, "_instance");
        if (instF)
        {
          void *am = nullptr;
          Il2cpp::GetFieldStaticValue(instF, &am);
          if (am)
          {
            // _engraveData at offset 0x88 (class ptr)
            void **edPtr = reinterpret_cast<void **>(reinterpret_cast<uint8_t *>(am) + 0x88);
            void *engraveData = *edPtr;
            if (engraveData)
            {
              // EngraveLevelList (List<int32>) at offset 0x18 (class ptr)
              void **listPtr = reinterpret_cast<void **>(reinterpret_cast<uint8_t *>(engraveData) + 0x18);
              void *listObj = *listPtr;
              if (listObj)
              {
                void **itemsPtr = reinterpret_cast<void **>(reinterpret_cast<uint8_t *>(listObj) + 0x10);
                int *sizePtr = reinterpret_cast<int *>(reinterpret_cast<uint8_t *>(listObj) + 0x18);
                _Il2CppArray *arr = reinterpret_cast<_Il2CppArray *>(*itemsPtr);
                int cnt = *sizePtr;
                if (arr && cnt > 0)
                {
                  // int32 elements at arr+0x20, each 4 bytes
                  int *elem0 = reinterpret_cast<int *>(reinterpret_cast<uint8_t *>(arr) + 0x20);
                  if (cnt > 0)
                    elem0[0] = 10000;
                  if (cnt > 1)
                    elem0[1] = 10000;
                  for (int i = 2; i < cnt; i++)
                    elem0[i] = 1000;
                  LOGI("ExtraCheat: EngraveLevelList set %d items", cnt);
                  total++;
                }
              }
            }
          }
        }
      }
    }

    // --- 3. StarlightBlessData.set_BuffCount ---
    {
      Il2CppClass *klass = Il2cpp::FindClass("DarkIdle2.StarlightBlessData");
      if (klass)
      {
        MethodInfo *setBC = Il2cpp::GetClassMethod(klass, "set_BuffCount", 1);
        if (setBC && setBC->methodPointer)
        {
          ScanCtx ctx = {klass, 0};
          il2cpp_gc_foreach_heap([](void *obj, void *ud)
                                 {
            auto *c = reinterpret_cast<ScanCtx*>(ud);
            if (SafeGetClass(obj) != c->targetClass) return;
            MethodInfo *m = reinterpret_cast<MethodInfo*>(c->count); // hack: store method in count field
            int val = 2000000000;
            void *params[] = {&val};
            Il2CppException *exc = nullptr;
            il2cpp_runtime_invoke(m, obj, params, &exc); }, &ctx);
          LOGI("ExtraCheat: StarlightBlessData BuffCount set");
          total++;
        }
      }
    }

    // --- 4. ArtifactSaveData: IsActive=true, Grade=20 ---
    {
      Il2CppClass *klass = Il2cpp::FindClass("DarkIdle2.Artifact.ArtifactSaveData");
      if (klass)
      {
        MethodInfo *setActive = Il2cpp::GetClassMethod(klass, "set_IsActive", 1);
        MethodInfo *setGrade = Il2cpp::GetClassMethod(klass, "set_Grade", 1);
        if (setActive && setGrade)
        {
          ScanCtx ctx = {klass, 0};
          il2cpp_gc_foreach_heap([](void *obj, void *ud)
                                 {
            auto *c = reinterpret_cast<ScanCtx*>(ud);
            if (SafeGetClass(obj) != c->targetClass) return;
            bool v = true; int g = 20;
            void *p1[] = {&v}; void *p2[] = {&g};
            Il2CppException *exc = nullptr;
            il2cpp_runtime_invoke(reinterpret_cast<MethodInfo*>(c->count), obj, p1, &exc); }, &ctx);
          LOGI("ExtraCheat: ArtifactSaveData set");
          total++;
        }
      }
    }

    // --- 5. ItemData.set_reforge(50) ---
    {
      Il2CppClass *klass = Il2cpp::FindClass("DarkIdle2.ItemData");
      if (klass)
      {
        MethodInfo *setReforge = Il2cpp::GetClassMethod(klass, "set_reforge", 1);
        if (setReforge && setReforge->methodPointer)
        {
          int val = 50;
          void *params[] = {&val};
          ScanCtx ctx = {klass, 0};
          il2cpp_gc_foreach_heap([](void *obj, void *ud)
                                 {
            auto *c = reinterpret_cast<ScanCtx*>(ud);
            if (SafeGetClass(obj) != c->targetClass) return;
            Il2CppException *exc = nullptr;
            il2cpp_runtime_invoke(reinterpret_cast<MethodInfo*>(c->count), obj, reinterpret_cast<void**>(c->targetClass), &exc); }, &ctx);
          LOGI("ExtraCheat: ItemData reforge set");
          total++;
        }
      }
    }

    LOGI("ExtraCheat: DONE! %d features executed", total);
  }

  static void InstallHook()
  {
    LOGI("ExtraCheat: ready");
  }
} // namespace ExtraCheat

namespace SpeedHack
{
  static MethodInfo *setTimeScaleMethod = nullptr;
  static float lastAppliedScale = -1.0f;

  static bool Resolve()
  {
    if (setTimeScaleMethod && setTimeScaleMethod->methodPointer)
      return true;

    setTimeScaleMethod = Il2cpp::FindMethod("UnityEngine.Time", "set_timeScale", 1);
    if (!setTimeScaleMethod || !setTimeScaleMethod->methodPointer)
    {
      LOGE("Speed: UnityEngine.Time.set_timeScale not found");
      return false;
    }

    LOGI("Speed: set_timeScale methodPointer=%p", setTimeScaleMethod->methodPointer);
    return true;
  }

  static void Apply(float scale)
  {
    if (!Resolve() || !il2cpp_runtime_invoke)
      return;

    if (scale < 1.0f)
      scale = 1.0f;
    if (scale > 10.0f)
      scale = 10.0f;

    void *params[1] = {&scale};
    Il2CppException *exc = nullptr;
    il2cpp_runtime_invoke(setTimeScaleMethod, nullptr, params, &exc);
    if (exc)
    {
      LOGE("Speed: set_timeScale threw exception");
      return;
    }

    lastAppliedScale = scale;
  }

  static void SetEnabled(bool enable)
  {
    g_SpeedEnabled = enable;
    LOGI("Speed: %s", enable ? "enabled" : "disabled");
    Apply(enable ? g_TimeScaleValue : 1.0f);
  }

  static void PerFrame()
  {
    float desired = g_SpeedEnabled ? g_TimeScaleValue : 1.0f;
    if (lastAppliedScale < 0.0f || desired != lastAppliedScale)
      Apply(desired);
  }
} // namespace SpeedHack

namespace UnlockQuest
{
  static bool InstallHook() { return false; }
  static void SetEnabled(bool enable) { g_UnlockQuestEnabled = enable; }
}

namespace OneHit
{
  static bool InstallHook() { return false; }
  static void SetEnabled(bool enable) { g_OneHitEnabled = enable; }
}

// ============================================================
// Add Ruby: ShopHelper là MonoBehaviour → scan heap mỗi lần trigger
// ============================================================
namespace RubyAdd
{
  static MethodInfo *giveRubyMethod = nullptr;

  static bool EnsureMethod()
  {
    if (giveRubyMethod && giveRubyMethod->methodPointer)
      return true;
    giveRubyMethod = Il2cpp::FindMethod("BPA.Outgame2.UI.Shop.ShopHelper", "GiveRuby", 0);
    if (!giveRubyMethod || !giveRubyMethod->methodPointer)
    {
      LOGE("RubyAdd: GiveRuby not found");
      return false;
    }
    LOGI("RubyAdd: GiveRuby=%p", giveRubyMethod->methodPointer);
    return true;
  }

  static Il2CppObject *FindShopHelper()
  {
    Il2CppClass *klass = Il2cpp::FindClass("BPA.Outgame2.UI.Shop.ShopHelper");
    if (!klass)
      return nullptr;

    struct Ctx
    {
      Il2CppClass *target;
      void *found;
    };
    Ctx ctx = {klass, nullptr};
    il2cpp_gc_foreach_heap([](void *obj, void *ud)
                           {
      auto *c = reinterpret_cast<Ctx *>(ud);
      if (SafeGetClass(obj) == c->target)
        c->found = obj; }, &ctx);
    return reinterpret_cast<Il2CppObject *>(ctx.found);
  }

  static void Trigger(int count)
  {
    if (count < 1)
      count = 1;
    if (count > 9999)
      count = 9999;

    if (!EnsureMethod())
      return;

    // Scan lại mỗi lần trigger vì MonoBehaviour chỉ có khi Shop mở
    Il2CppObject *inst = FindShopHelper();
    if (!inst)
    {
      LOGE("RubyAdd: ShopHelper not found (is Shop open?)");
      return;
    }
    LOGI("RubyAdd: instance=%p, GiveRuby x%d...", inst, count);

    for (int i = 0; i < count; i++)
    {
      Il2CppException *exc = nullptr;
      il2cpp_runtime_invoke(giveRubyMethod, inst, nullptr, &exc);
      if (exc)
      {
        LOGE("RubyAdd: exception at %d", i);
        break;
      }
    }
    LOGI("RubyAdd: done");
  }
} // namespace RubyAdd

void SetupImGuiCommonStyleAndFont()
{
  ImGuiIO &io = ImGui::GetIO();
  ImGui::StyleColorsDark();
  ImGuiStyle &style = ImGui::GetStyle();
  style.WindowTitleAlign = ImVec2(0.5f, 0.5f);
  style.FrameRounding = 5.0f;
  style.GrabRounding = 4.0f;
  style.WindowRounding = 8.0f;
  style.ChildRounding = 5.0f;
  style.ScrollbarSize = 22.0f;
  style.ScrollbarRounding = 11.0f;
  style.FrameBorderSize = 1.0f;
  style.WindowBorderSize = 1.0f;
  style.PopupBorderSize = 1.0f;
  style.PopupRounding = 5.0f;

  style.Colors[ImGuiCol_WindowBg] = ImVec4(0.06f, 0.06f, 0.09f, 0.96f);
  style.Colors[ImGuiCol_TitleBg] = ImVec4(0.20f, 0.25f, 0.43f, 1.00f);
  style.Colors[ImGuiCol_TitleBgActive] = ImVec4(0.29f, 0.37f, 0.65f, 1.00f);
  style.Colors[ImGuiCol_FrameBg] = ImVec4(0.09f, 0.11f, 0.17f, 1.00f);
  style.Colors[ImGuiCol_FrameBgHovered] = ImVec4(0.14f, 0.17f, 0.26f, 1.00f);
  style.Colors[ImGuiCol_FrameBgActive] = ImVec4(0.20f, 0.25f, 0.38f, 1.00f);
  style.Colors[ImGuiCol_Button] = ImVec4(0.20f, 0.25f, 0.43f, 1.00f);
  style.Colors[ImGuiCol_ButtonHovered] = ImVec4(0.28f, 0.35f, 0.60f, 1.00f);
  style.Colors[ImGuiCol_ButtonActive] = ImVec4(0.35f, 0.44f, 0.75f, 1.00f);
  style.Colors[ImGuiCol_CheckMark] = ImVec4(0.29f, 0.37f, 0.65f, 1.00f);
  style.Colors[ImGuiCol_SliderGrab] = ImVec4(0.29f, 0.37f, 0.65f, 1.00f);
  style.Colors[ImGuiCol_SliderGrabActive] = ImVec4(0.35f, 0.44f, 0.75f, 1.00f);
  style.Colors[ImGuiCol_Header] = ImVec4(0.20f, 0.25f, 0.43f, 1.00f);
  style.Colors[ImGuiCol_HeaderHovered] = ImVec4(0.28f, 0.35f, 0.60f, 1.00f);
  style.Colors[ImGuiCol_HeaderActive] = ImVec4(0.35f, 0.44f, 0.75f, 1.00f);
  style.Colors[ImGuiCol_Separator] = ImVec4(0.25f, 0.32f, 0.55f, 0.50f);
  style.Colors[ImGuiCol_SeparatorHovered] = ImVec4(0.29f, 0.37f, 0.65f, 0.70f);
  style.Colors[ImGuiCol_SeparatorActive] = ImVec4(0.35f, 0.44f, 0.75f, 1.00f);
  style.Colors[ImGuiCol_Text] = ImVec4(0.94f, 0.94f, 0.96f, 1.00f);
  style.Colors[ImGuiCol_TextDisabled] = ImVec4(0.50f, 0.53f, 0.65f, 1.00f);
  style.Colors[ImGuiCol_Border] = ImVec4(0.25f, 0.32f, 0.55f, 0.80f);
  style.Colors[ImGuiCol_ResizeGrip] = ImVec4(0.20f, 0.25f, 0.43f, 0.40f);
  style.Colors[ImGuiCol_ResizeGripHovered] = ImVec4(0.28f, 0.35f, 0.60f, 0.60f);
  style.Colors[ImGuiCol_ResizeGripActive] = ImVec4(0.35f, 0.44f, 0.75f, 0.80f);
  style.Colors[ImGuiCol_ScrollbarBg] = ImVec4(0.06f, 0.06f, 0.09f, 0.50f);
  style.Colors[ImGuiCol_ScrollbarGrab] = ImVec4(0.20f, 0.25f, 0.43f, 0.50f);
  style.Colors[ImGuiCol_ScrollbarGrabHovered] = ImVec4(0.28f, 0.35f, 0.60f, 0.70f);
  style.Colors[ImGuiCol_ScrollbarGrabActive] = ImVec4(0.35f, 0.44f, 0.75f, 0.90f);
  SetupMenuFonts(io, 28.0f);
  ImGui::GetStyle().ScaleAllSizes(1.0f);
  CaptureBaseMenuStyle();
}

static bool SocialIconButton(const char *id, const char *icon, ImU32 fillColor, const char *url)
{
  ImGui::PushID(id);
  const float size = 62.0f;
  ImVec2 pos = ImGui::GetCursorScreenPos();
  bool pressed = ImGui::InvisibleButton("##social", ImVec2(size, size));
  ImDrawList *draw = ImGui::GetWindowDrawList();
  ImVec2 center(pos.x + size * 0.5f, pos.y + size * 0.5f);
  ImU32 ringColor = ImGui::IsItemHovered() ? IM_COL32(255, 255, 255, 245) : IM_COL32(255, 255, 255, 150);
  draw->AddCircleFilled(center, size * 0.48f, fillColor, 36);
  draw->AddCircle(center, size * 0.48f, ringColor, 36, 2.0f);
  ImFont *font = ImGui::GetFont();
  float iconFontSize = font->FontSize * 0.98f;
  ImVec2 iconSize = font->CalcTextSizeA(iconFontSize, FLT_MAX, 0.0f, icon);
  draw->AddText(font,
                iconFontSize,
                ImVec2(center.x - iconSize.x * 0.5f, center.y - iconSize.y * 0.52f),
                IM_COL32(255, 255, 255, 255),
                icon);

  if (pressed)
    OpenAndroidUrl(url);
  ImGui::PopID();
  return pressed;
}

static void DrawMenuHeader()
{
  ImDrawList *draw = ImGui::GetWindowDrawList();
  ImVec2 start = ImGui::GetCursorScreenPos();
  float width = ImGui::GetContentRegionAvail().x;
  const float headerHeight = 34.0f;

  const char *title = "Menu Modder By V7";
  ImFont *font = ImGui::GetFont();
  float titleFontSize = font->FontSize * 0.78f;
  ImVec2 titleSize = font->CalcTextSizeA(titleFontSize, FLT_MAX, 0.0f, title);
  ImVec2 titlePos(start.x + (width - titleSize.x) * 0.5f, start.y + 7.0f);
  draw->AddText(font, titleFontSize, titlePos, IM_COL32(238, 232, 242, 255), title);

  ImGui::SetCursorScreenPos(ImVec2(start.x, start.y + headerHeight));
}

static void DrawMenuSocialFooter()
{
  float width = ImGui::GetContentRegionAvail().x;
  const float iconSize = 62.0f;
  float spacing = (width - iconSize * 4.0f) / 5.0f;
  if (spacing < 8.0f)
    spacing = 8.0f;
  float totalWidth = iconSize * 4.0f + spacing * 3.0f;
  float startX = ImGui::GetCursorScreenPos().x + (width - totalWidth) * 0.5f;
  if (startX < ImGui::GetCursorScreenPos().x)
    startX = ImGui::GetCursorScreenPos().x;

  ImGui::SetCursorScreenPos(ImVec2(startX, ImGui::GetCursorScreenPos().y));
  SocialIconButton("youtube", ICON_FA_YOUTUBE, IM_COL32(230, 30, 42, 235), "https://www.youtube.com/@SilverV7NTV");
  ImGui::SameLine(0.0f, spacing);
  SocialIconButton("telegram", ICON_FA_TELEGRAM, IM_COL32(36, 160, 220, 235), "https://t.me/NTVV7");
  ImGui::SameLine(0.0f, spacing);
  SocialIconButton("facebook", ICON_FA_FACEBOOK, IM_COL32(40, 95, 210, 235), "https://www.facebook.com/HelloIAMNTV.Cute");
  ImGui::SameLine(0.0f, spacing);
  SocialIconButton("website", ICON_FA_GLOBE, IM_COL32(45, 185, 120, 235), "https://www.silverv7.online/");
}

#include <thread>
#include <fstream>
#include <sstream>
#include <algorithm>

static int g_DumpStatus = 0; // 0: Idle, 1: Dumping/Writing, 2: Done, 3: Failed
static std::string g_DumpStatusMsg = "";

extern Il2CppDomain *(*il2cpp_domain_get)();
extern Il2CppClass *(*il2cpp_class_get_parent)(Il2CppClass *klass);
extern int (*il2cpp_class_get_flags)(Il2CppClass *klass);

static std::string CleanTypeName(const char *name)
{
  if (!name)
    return "void";
  std::string s(name);
  if (s == "System.Int32")
    return "int";
  if (s == "System.Int64")
    return "long";
  if (s == "System.UInt32")
    return "uint";
  if (s == "System.UInt64")
    return "ulong";
  if (s == "System.Int16")
    return "short";
  if (s == "System.UInt16")
    return "ushort";
  if (s == "System.Byte")
    return "byte";
  if (s == "System.SByte")
    return "sbyte";
  if (s == "System.Single")
    return "float";
  if (s == "System.Double")
    return "double";
  if (s == "System.Boolean")
    return "bool";
  if (s == "System.String")
    return "string";
  if (s == "System.Char")
    return "char";
  if (s == "System.Object")
    return "object";
  if (s == "System.Void")
    return "void";
  return s;
}

static std::string GetFriendlyTypeName(Il2CppType *type)
{
  if (!type)
    return "Unknown";

  const char *typeName = Il2cpp::GetTypeName(type);
  std::string cleanName = CleanTypeName(typeName);
  Il2CppClass *klass = Il2cpp::GetClassFromType(type);

  if (klass && Il2cpp::GetClassIsEnum(klass))
    return "Enum<" + CleanTypeName(Il2cpp::GetClassName(klass)) + ">";

  switch (type->type)
  {
  case IL2CPP_TYPE_BOOLEAN:
    return "bool";
  case IL2CPP_TYPE_CHAR:
    return "char";
  case IL2CPP_TYPE_I1:
    return "sbyte";
  case IL2CPP_TYPE_U1:
    return "byte";
  case IL2CPP_TYPE_I2:
    return "short";
  case IL2CPP_TYPE_U2:
    return "ushort";
  case IL2CPP_TYPE_I4:
    return "int";
  case IL2CPP_TYPE_U4:
    return "uint";
  case IL2CPP_TYPE_I8:
    return "long";
  case IL2CPP_TYPE_U8:
    return "ulong";
  case IL2CPP_TYPE_R4:
    return "float";
  case IL2CPP_TYPE_R8:
    return "double";
  case IL2CPP_TYPE_STRING:
    return "string";
  case IL2CPP_TYPE_ARRAY:
  case IL2CPP_TYPE_SZARRAY:
    return "Array";
  default:
    break;
  }

  if (cleanName.find("System.Collections.Generic.Dictionary") == 0 ||
      cleanName.find("Dictionary<") != std::string::npos)
    return "Dictionary";
  if (cleanName.find("System.Collections.Generic.List") == 0 ||
      cleanName.find("List<") != std::string::npos)
    return "List";
  if (cleanName.find("System.Collections.Generic.HashSet") == 0 ||
      cleanName.find("HashSet<") != std::string::npos)
    return "HashSet";

  if (klass && Il2cpp::GetClassIsValueType(klass))
    return "struct";

  return cleanName;
}

static std::string FormatMethodSignature(const std::string &retType, const std::string &name, MethodInfo *method)
{
  std::string sig = retType + " " + name + "(";
  int paramCount = Il2cpp::GetMethodParamCount(method);
  for (int p = 0; p < paramCount; p++)
  {
    auto paramType = Il2cpp::GetMethodParam(method, p);
    const char *paramTypeName = paramType ? Il2cpp::GetTypeName(paramType) : "System.Void";
    const char *paramName = Il2cpp::GetMethodParamName(method, p);

    sig += CleanTypeName(paramTypeName);
    if (paramName && paramName[0])
    {
      sig += " ";
      sig += paramName;
    }
    if (p < paramCount - 1)
    {
      sig += ", ";
    }
  }
  sig += ")";
  return sig;
}

static std::string FormatFieldSignature(const std::string &type, const std::string &name, size_t offset)
{
  char buf[64];
  snprintf(buf, sizeof(buf), " // Offset: 0x%X", (unsigned int)offset);
  return type + " " + name + ";" + buf;
}

static std::string BuildClassFullName(const FlatResultEntry &item)
{
  return item.ns.empty() ? item.name : (item.ns + "." + item.name);
}

static void SetSelectedClassState(int index, bool saveNow)
{
  if (index < 0 || index >= (int)g_FilteredList.size())
    return;
  std::string previousClass = g_LastSelectedClassFullName;
  g_SelectedFlatIdx = index;
  g_LastSelectedClassFullName = BuildClassFullName(g_FilteredList[(size_t)index]);
  if (previousClass != g_LastSelectedClassFullName)
    g_LastSelectedMethodName.clear();
  if (saveNow)
    SaveConfig();
}

static void RestoreSavedClassSelection()
{
  if (!g_RestoreSelectionPending || g_LastSelectedClassFullName.empty() || g_FilteredList.empty())
    return;

  for (size_t i = 0; i < g_FilteredList.size(); ++i)
  {
    if (BuildClassFullName(g_FilteredList[i]) == g_LastSelectedClassFullName)
    {
      g_SelectedFlatIdx = (int)i;
      g_RestoreSelectionPending = false;
      return;
    }
  }
}

static std::string BuildSearchTabTitle(const SearchTabState &tab)
{
  std::string text;
  if (tab.type == SearchTabState::TYPE_INSPECT_OBJECT)
  {
    if (tab.inspectPtr)
    {
      Il2CppClass *klass = SafeGetClass(tab.inspectPtr);
      const char *className = klass ? Il2cpp::GetClassName(klass) : "Unknown";
      text = std::string("Inspect ") + className;
    }
    else
    {
      text = "Inspecting";
    }
  }
  else if (tab.type == SearchTabState::TYPE_INSPECT_RAW)
  {
    text = "Inspect " + tab.inspectLabel;
  }
  else if (!tab.selectedMethod.empty())
  {
    text = tab.selectedMethod;
  }
  else if (tab.selectedFlatIdx >= 0 && tab.selectedFlatIdx < (int)tab.results.size())
  {
    text = tab.results[tab.selectedFlatIdx].name;
    size_t dotPos = text.rfind('.');
    if (dotPos != std::string::npos)
      text = text.substr(dotPos + 1);
  }
  else
  {
    if (tab.filter.empty())
    {
      std::string imgName = "Search";
      if (tab.selectedImageIdx >= 0 && tab.selectedImageIdx < (int)g_MetadataCache.size())
      {
        imgName = g_MetadataCache[tab.selectedImageIdx].name;
        if (imgName.size() >= 4 && imgName.compare(imgName.size() - 4, 4, ".dll") == 0)
        {
          imgName = imgName.substr(0, imgName.size() - 4);
        }
        else if (imgName.size() >= 4 && imgName.compare(imgName.size() - 4, 4, ".DLL") == 0)
        {
          imgName = imgName.substr(0, imgName.size() - 4);
        }
      }
      text = imgName;
    }
    else
    {
      text = tab.filter;
    }
  }

  size_t maxLen = (tab.type == SearchTabState::TYPE_SEARCH) ? 12 : 28;
  size_t trunLen = (tab.type == SearchTabState::TYPE_SEARCH) ? 9 : 25;
  if (text.size() > maxLen)
    text = text.substr(0, trunLen) + "...";
  return text;
}

static void EnsureSearchTabs()
{
  if (!g_SearchTabs.empty())
    return;
  SearchTabState tab;
  tab.title = "";
  tab.selectedImageIdx = g_SelectedImageIdx;
  tab.searchAllImages = g_SearchAllImages;
  tab.filter = g_FilterSearch;
  tab.filterType = g_FilterType;
  tab.filterMethodOption = g_FilterMethodOption;
  tab.splitLayout = g_SplitLayout;
  tab.selectedFlatIdx = g_SelectedFlatIdx;
  tab.selectedMethod = g_LastSelectedMethodName;
  tab.results = g_FilteredList;

  tab.lastSelectedClassFullName = g_LastSelectedClassFullName;

  g_SearchTabs.push_back(tab);
  g_ActiveSearchTab = 0;
}

static void SaveActiveSearchTabState()
{
  EnsureSearchTabs();
  if (g_ActiveSearchTab < 0 || g_ActiveSearchTab >= (int)g_SearchTabs.size())
    return;
  SearchTabState &tab = g_SearchTabs[g_ActiveSearchTab];

  if (tab.type == SearchTabState::TYPE_SEARCH)
  {
    tab.selectedImageIdx = g_SelectedImageIdx;
    tab.searchAllImages = g_SearchAllImages;
    tab.filter = g_FilterSearch;
    tab.filterType = g_FilterType;
    tab.filterMethodOption = g_FilterMethodOption;
    tab.splitLayout = g_SplitLayout;
    tab.selectedFlatIdx = g_SelectedFlatIdx;
    tab.selectedMethod = g_LastSelectedMethodName;
    tab.results = g_FilteredList;
    tab.lastSelectedClassFullName = g_LastSelectedClassFullName;
  }

  tab.title = BuildSearchTabTitle(tab);
}

static void LoadSearchTabState(int index)
{
  EnsureSearchTabs();
  if (index < 0 || index >= (int)g_SearchTabs.size())
    return;
  g_ActiveSearchTab = index;
  const SearchTabState &tab = g_SearchTabs[index];

  if (tab.type == SearchTabState::TYPE_SEARCH)
  {
    g_SelectedImageIdx = tab.selectedImageIdx;
    g_SearchAllImages = tab.searchAllImages;
    snprintf(g_FilterSearch, sizeof(g_FilterSearch), "%s", tab.filter.c_str());
    g_PatternSearch[0] = '\0';
    g_LastPatternSearch[0] = '\0';
    snprintf(g_LastFilterSearch, sizeof(g_LastFilterSearch), "%s", g_FilterSearch);
    g_FilterType = tab.filterType;
    g_FilterMethodOption = tab.filterMethodOption;
    g_SplitLayout = tab.splitLayout;
    g_SelectedFlatIdx = tab.selectedFlatIdx;
    g_LastSelectedMethodName = tab.selectedMethod;
    g_FilteredList = tab.results;
    g_FilterNeedsUpdate = g_FilteredList.empty() && !tab.filter.empty();
    g_LastSelectedClassFullName = tab.lastSelectedClassFullName;
  }
}

static void AddSearchTab()
{
  SaveActiveSearchTabState();
  SearchTabState tab;
  tab.title = "";
  tab.selectedImageIdx = g_SelectedImageIdx;
  tab.searchAllImages = g_SearchAllImages;
  tab.filterType = g_FilterType;
  tab.filterMethodOption = g_FilterMethodOption;
  tab.splitLayout = g_SplitLayout;
  tab.selectedFlatIdx = -1;
  tab.selectedMethod = "";
  g_SearchTabs.push_back(tab);
  LoadSearchTabState((int)g_SearchTabs.size() - 1);
  g_RequestSearchTabSelection = true;
  SaveConfig();
}

static std::string BuildClassDeclaration(const FlatResultEntry &item)
{
  std::string text = "public ";
  if (item.isEnum)
    text += "enum ";
  else if (item.isStruct)
    text += "struct ";
  else
    text += "class ";
  text += item.name;
  if (!item.parentName.empty())
  {
    text += " : " + item.parentName;
  }
  return text;
}

static bool RenderInspectorFullViewToggleButton(const char *id, const ImVec2 &size = ImVec2(32.0f, 0.0f))
{
  std::string label = std::string(g_ToolsInspectorFullView ? "Compact##" : "[]##") + id;
  if (ImGui::Button(label.c_str(), size))
  {
    g_ToolsInspectorFullView = !g_ToolsInspectorFullView;
    SaveConfig();
    return true;
  }
  if (ImGui::IsItemHovered())
    ImGui::SetTooltip(g_ToolsInspectorFullView ? "Compact view" : "Full inspector view");
  return false;
}

static void RenderSearchTabsBar()
{
  EnsureSearchTabs();
  if (ImGui::ArrowButton("##search_tab_menu", ImGuiDir_Down))
    ImGui::OpenPopup("##SearchTabMenu");
  if (ImGui::IsItemHovered())
    ImGui::SetTooltip("Search tabs");
  ImGui::SetNextWindowSizeConstraints(ImVec2(240.0f, 0.0f), ImVec2(450.0f, 600.0f));
  ImGui::PushStyleVar(ImGuiStyleVar_PopupBorderSize, 1.0f);
  ImGui::PushStyleVar(ImGuiStyleVar_PopupRounding, 4.0f);
  if (ImGui::BeginPopup("##SearchTabMenu"))
  {
    ImGui::PushStyleColor(ImGuiCol_HeaderHovered, ImVec4(0.28f, 0.35f, 0.60f, 1.0f));
    ImGui::PushStyleColor(ImGuiCol_HeaderActive, ImVec4(0.35f, 0.44f, 0.75f, 1.0f));
    ImGui::PushStyleColor(ImGuiCol_Header, ImVec4(0.20f, 0.25f, 0.43f, 1.0f));

    for (int i = 0; i < (int)g_SearchTabs.size(); ++i)
    {
      std::string label = g_SearchTabs[i].title;
      if (g_SearchTabs[i].filter.empty() && g_SearchTabs[i].selectedMethod.empty() && g_SearchTabs[i].selectedFlatIdx < 0)
        label = BuildSearchTabTitle(g_SearchTabs[i]);
      else if (label.empty())
        label = BuildSearchTabTitle(g_SearchTabs[i]);

      std::string selectableNameWithId = label + "##search_tab_sel_" + std::to_string(i);
      if (ImGui::Selectable(selectableNameWithId.c_str(), i == g_ActiveSearchTab))
      {
        SaveActiveSearchTabState();
        LoadSearchTabState(i);
        g_RequestSearchTabSelection = true;
        SaveConfig();
      }
    }

    ImGui::PopStyleColor(3);
    ImGui::EndPopup();
    ImGui::PopStyleVar(2);
  }

  ImGui::SameLine();
  if (ImGui::Button("+##new_tools_search_tab", ImVec2(ImGui::GetFrameHeight(), 0)))
  {
    AddSearchTab();
  }
  if (ImGui::IsItemHovered())
    ImGui::SetTooltip("New search tab");

  ImGui::SameLine();
  ImGui::PushStyleColor(ImGuiCol_Tab, ImVec4(0.11f, 0.13f, 0.21f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_TabHovered, ImVec4(0.20f, 0.25f, 0.40f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_TabActive, ImVec4(0.29f, 0.37f, 0.65f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_TabUnfocused, ImVec4(0.11f, 0.13f, 0.21f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_TabUnfocusedActive, ImVec4(0.29f, 0.37f, 0.65f, 1.00f));

  if (ImGui::BeginTabBar("##SearchTabsTabBar", ImGuiTabBarFlags_FittingPolicyScroll))
  {
    int closeTabIdx = -1;
    for (int i = 0; i < (int)g_SearchTabs.size(); ++i)
    {
      std::string label = g_SearchTabs[i].title;
      if (g_SearchTabs[i].filter.empty() && g_SearchTabs[i].selectedMethod.empty() && g_SearchTabs[i].selectedFlatIdx < 0)
        label = BuildSearchTabTitle(g_SearchTabs[i]);
      else if (label.empty())
        label = BuildSearchTabTitle(g_SearchTabs[i]);

      std::string tabNameWithId = label + "###search_tab_" + std::to_string(i);
      bool active = i == g_ActiveSearchTab;

      ImGuiTabItemFlags flags = 0;
      if (g_RequestSearchTabSelection && active)
        flags |= ImGuiTabItemFlags_SetSelected;

      bool open = true;
      ImGui::PushID(i);
      if (ImGui::BeginTabItem(tabNameWithId.c_str(), &open, flags))
      {
        if (!g_RequestSearchTabSelection && i != g_ActiveSearchTab)
        {
          SaveActiveSearchTabState();
          LoadSearchTabState(i);
          SaveConfig();
        }
        ImGui::EndTabItem();
      }
      ImGui::PopID();

      if (!open)
      {
        closeTabIdx = i;
      }
    }
    ImGui::EndTabBar();

    g_RequestSearchTabSelection = false;

    if (closeTabIdx != -1)
    {
      if (g_SearchTabs.size() > 1)
      {
        const auto &tab = g_SearchTabs[closeTabIdx];
        void *ptrToUnpin = (tab.type == SearchTabState::TYPE_INSPECT_OBJECT) ? tab.inspectPtr : nullptr;
        if (tab.type != SearchTabState::TYPE_SEARCH && tab.inspectPtr)
        {
          g_TabHistories.erase(tab.inspectPtr);
        }
        g_SearchTabs.erase(g_SearchTabs.begin() + closeTabIdx);
        if (ptrToUnpin)
        {
          UnpinInspectedObject(ptrToUnpin);
        }
        if (closeTabIdx < g_ActiveSearchTab)
        {
          g_ActiveSearchTab--;
        }
        else if (g_ActiveSearchTab >= (int)g_SearchTabs.size())
        {
          g_ActiveSearchTab = (int)g_SearchTabs.size() - 1;
        }
        LoadSearchTabState(g_ActiveSearchTab);
        g_RequestSearchTabSelection = true;
        SaveConfig();
      }
      else
      {
        void *ptrToUnpin = (g_SearchTabs[closeTabIdx].type == SearchTabState::TYPE_INSPECT_OBJECT) ? g_SearchTabs[closeTabIdx].inspectPtr : nullptr;
        g_SearchTabs[closeTabIdx].type = SearchTabState::TYPE_SEARCH;
        g_SearchTabs[closeTabIdx].inspectPtr = nullptr;
        g_SearchTabs[closeTabIdx].title = "";
        g_SearchTabs[closeTabIdx].filter = "";
        g_SearchTabs[closeTabIdx].results.clear();
        g_SearchTabs[closeTabIdx].selectedFlatIdx = -1;
        g_SearchTabs[closeTabIdx].selectedMethod = "";
        g_SearchTabs[closeTabIdx].lastSelectedClassFullName = "";
        if (ptrToUnpin)
        {
          UnpinInspectedObject(ptrToUnpin);
        }
        LoadSearchTabState(closeTabIdx);
        g_RequestSearchTabSelection = true;
        SaveConfig();
      }
    }
  }
  ImGui::PopStyleColor(5);
}

static void RenderCopyPopupForLastItem(const char *id,
                                       const std::string &title,
                                       const std::vector<std::pair<std::string, std::string>> &items)
{
  static std::string heldId;
  static std::string openedId;
  static double holdStart = 0.0;

  std::string popupId = std::string("##copy_popup_") + id;
  bool hovered = ImGui::IsItemHovered(ImGuiHoveredFlags_AllowWhenBlockedByPopup);
  bool rightClicked = ImGui::IsItemClicked(ImGuiMouseButton_Right);
  bool holding = hovered && ImGui::IsMouseDown(ImGuiMouseButton_Left);
  double now = ImGui::GetTime();

  if (rightClicked)
  {
    ImGui::OpenPopup(popupId.c_str());
    openedId = id;
  }
  else if (holding)
  {
    if (heldId != id)
    {
      heldId = id;
      holdStart = now;
    }
    else if (openedId != id && now - holdStart >= 0.55)
    {
      ImGui::OpenPopup(popupId.c_str());
      openedId = id;
    }
  }
  else if (heldId == id && !ImGui::IsMouseDown(ImGuiMouseButton_Left))
  {
    heldId.clear();
  }

  // Push style for beautiful border and rounding
  ImGui::PushStyleColor(ImGuiCol_Border, ImVec4(0.30f, 0.25f, 0.35f, 1.0f)); // Subtle dark violet/gray border
  ImGui::PushStyleVar(ImGuiStyleVar_PopupRounding, 6.0f);
  ImGui::PushStyleVar(ImGuiStyleVar_PopupBorderSize, 1.0f);
  ImGui::PushStyleVar(ImGuiStyleVar_WindowPadding, ImVec2(12.0f, 8.0f)); // Spacing inside popup

  bool showPopup = ImGui::BeginPopup(popupId.c_str());

  ImGui::PopStyleVar(3);
  ImGui::PopStyleColor();

  if (showPopup)
  {
    // Center the "——  Copy  ——" text
    float availW = ImGui::GetContentRegionAvail().x;
    float textW = ImGui::CalcTextSize("——  Copy  ——").x;
    if (availW > textW)
      ImGui::SetCursorPosX(ImGui::GetCursorPosX() + (availW - textW) * 0.5f);
    ImGui::TextDisabled("——  Copy  ——");
    ImGui::Spacing();

    // Determine the button label based on title
    std::string buttonLabel = "Copy Name";
    std::string copyValue = "";

    if (title == "Copy Class")
    {
      buttonLabel = "Copy Class Name";
      for (const auto &item : items)
      {
        if (item.first == "Full Name")
        {
          copyValue = item.second;
          break;
        }
      }
      if (copyValue.empty() && !items.empty())
        copyValue = items[0].second;
    }
    else if (title == "Copy Method")
    {
      buttonLabel = "Copy Method Name";
      for (const auto &item : items)
      {
        if (item.first == "Name")
        {
          copyValue = item.second;
          break;
        }
      }
      if (copyValue.empty() && !items.empty())
        copyValue = items[0].second;
    }
    else if (title == "Copy Field")
    {
      buttonLabel = "Copy Field Name";
      for (const auto &item : items)
      {
        if (item.first == "Name")
        {
          copyValue = item.second;
          break;
        }
      }
      if (copyValue.empty() && !items.empty())
        copyValue = items[0].second;
    }
    else // default/fallback (e.g. Copy Result)
    {
      buttonLabel = title; // e.g. "Copy Result"
      if (!items.empty())
        copyValue = items[0].second;
    }

    // Render as a selectable menu item
    ImGui::PushStyleColor(ImGuiCol_HeaderHovered, ImVec4(0.25f, 0.18f, 0.32f, 1.0f)); // Hover highlight color
    ImGui::PushStyleColor(ImGuiCol_HeaderActive, ImVec4(0.35f, 0.24f, 0.42f, 1.0f));

    if (ImGui::Selectable(buttonLabel.c_str(), false))
    {
      CopyToClipboard(copyValue.c_str());
      ImGui::CloseCurrentPopup();
    }

    ImGui::PopStyleColor(2);
    ImGui::EndPopup();
  }
}

static std::string LuaQuote(const std::string &text)
{
  std::string out = "\"";
  for (char c : text)
  {
    if (c == '\\' || c == '"')
      out += '\\';
    out += c;
  }
  out += "\"";
  return out;
}

static std::string BuildRedirectScript(const std::string &sourceClass,
                                       const MethodEntry &sourceMethod,
                                       const std::string &targetClass,
                                       const MethodEntry &targetMethod)
{
  return "Hook.redirectDirect(" +
         LuaQuote(sourceClass) + ", " +
         LuaQuote(sourceMethod.name) + ", " +
         (sourceClass == targetClass
              ? LuaQuote(targetMethod.name)
              : LuaQuote(targetClass) + ", " + LuaQuote(targetMethod.name)) +
         ")";
}

static std::string LowerCopy(std::string text)
{
  std::transform(text.begin(), text.end(), text.begin(), ::tolower);
  return text;
}

static std::string GetReturnLuaFunction(const std::string &returnType)
{
  std::string type = LowerCopy(returnType);
  if (type == "bool" || type == "boolean" || type.find("system.boolean") != std::string::npos)
    return "returnBool";
  if (type == "string" || type.find("system.string") != std::string::npos)
    return "returnString";
  if (type == "long" || type == "int64" || type == "uint64" ||
      type.find("system.int64") != std::string::npos || type.find("system.uint64") != std::string::npos)
    return "returnLong";
  if (type == "float" || type == "single" || type.find("system.single") != std::string::npos)
    return "returnFloat";
  if (type == "double" || type.find("system.double") != std::string::npos)
    return "returnDouble";
  if (type == "int" || type == "int32" || type == "uint32" ||
      type == "short" || type == "uint16" || type == "byte" ||
      type.find("system.int32") != std::string::npos || type.find("system.uint32") != std::string::npos ||
      type.find("system.int16") != std::string::npos || type.find("system.uint16") != std::string::npos ||
      type.find("system.byte") != std::string::npos)
    return "returnInt";
  return "";
}

static std::string BuildReturnScript(const std::string &className,
                                     const MethodEntry &method,
                                     const std::string &luaFunc,
                                     const std::string &valueExpr)
{
  return "Hook." + luaFunc + "(" +
         LuaQuote(className) + ", " +
         LuaQuote(method.name) + ", " +
         valueExpr + ")";
}

static std::string GetReturnLuaFunctionFromPatchType(const std::string &patchType)
{
  if (patchType == "ReturnBool")
    return "returnBool";
  if (patchType == "ReturnInt32")
    return "returnInt";
  if (patchType == "ReturnInt64")
    return "returnLong";
  if (patchType == "ReturnFloat")
    return "returnFloat";
  if (patchType == "ReturnDouble")
    return "returnDouble";
  if (patchType == "ReturnString")
    return "returnString";
  return "";
}

static bool IsIntTypeName(const std::string &type)
{
  std::string t = LowerCopy(type);
  return t == "int" || t == "int32" || t == "uint32" || t == "short" || t == "uint16" ||
         t == "byte" || t == "sbyte" || t.find("system.int32") != std::string::npos ||
         t.find("system.uint32") != std::string::npos || t.find("system.int16") != std::string::npos ||
         t.find("system.uint16") != std::string::npos || t.find("system.byte") != std::string::npos;
}

static bool IsLongTypeName(const std::string &type)
{
  std::string t = LowerCopy(type);
  return t == "long" || t == "int64" || t == "uint64" ||
         t.find("system.int64") != std::string::npos || t.find("system.uint64") != std::string::npos;
}

static bool IsFloatTypeName(const std::string &type)
{
  std::string t = LowerCopy(type);
  return t == "float" || t == "single" || t.find("system.single") != std::string::npos;
}

static bool IsDoubleTypeName(const std::string &type)
{
  std::string t = LowerCopy(type);
  return t == "double" || t.find("system.double") != std::string::npos;
}

static bool IsBoolTypeName(const std::string &type)
{
  std::string t = LowerCopy(type);
  return t == "bool" || t == "boolean" || t.find("system.boolean") != std::string::npos;
}

static bool IsStringTypeName(const std::string &type)
{
  std::string t = LowerCopy(type);
  return t == "string" || t.find("system.string") != std::string::npos;
}

static bool IsTaskTypeName(const std::string &type)
{
  std::string t = LowerCopy(type);
  return t.find("system.threading.tasks.task") != std::string::npos ||
         t == "task" || t.find("task<") != std::string::npos;
}

static bool IsCallerObjectTypeName(const std::string &type)
{
  return !IsBoolTypeName(type) &&
         !IsIntTypeName(type) &&
         !IsLongTypeName(type) &&
         !IsFloatTypeName(type) &&
         !IsDoubleTypeName(type) &&
         !IsStringTypeName(type);
}

static std::string TrimCopy(const std::string &text)
{
  size_t start = 0;
  while (start < text.size() && std::isspace((unsigned char)text[start]))
    ++start;

  size_t end = text.size();
  while (end > start && std::isspace((unsigned char)text[end - 1]))
    --end;

  return text.substr(start, end - start);
}

static void AddUniqueScanCandidate(std::vector<std::string> &candidates, const std::string &name)
{
  std::string clean = TrimCopy(name);
  if (clean.empty())
    return;

  if (std::find(candidates.begin(), candidates.end(), clean) == candidates.end())
    candidates.push_back(clean);
}

static int CountGenericTopLevelArgs(const std::string &typeName)
{
  size_t open = typeName.find('<');
  size_t close = typeName.rfind('>');
  if (open == std::string::npos || close == std::string::npos || close <= open + 1)
    return 0;

  int depth = 0;
  int commas = 0;
  bool hasToken = false;
  for (size_t i = open + 1; i < close; ++i)
  {
    char c = typeName[i];
    if (c == '<')
    {
      ++depth;
      hasToken = true;
    }
    else if (c == '>')
    {
      if (depth > 0)
        --depth;
    }
    else if (c == ',' && depth == 0)
    {
      ++commas;
    }
    else if (!std::isspace((unsigned char)c))
    {
      hasToken = true;
    }
  }

  return hasToken ? commas + 1 : 0;
}

static std::vector<std::string> BuildScanClassCandidates(const std::string &rawTypeName)
{
  std::vector<std::string> candidates;
  std::string typeName = TrimCopy(rawTypeName);
  AddUniqueScanCandidate(candidates, typeName);

  while (!typeName.empty() && (typeName.back() == '&' || typeName.back() == '*'))
    typeName.pop_back();
  typeName = TrimCopy(typeName);
  AddUniqueScanCandidate(candidates, typeName);

  if (typeName.size() > 2 && typeName.compare(typeName.size() - 2, 2, "[]") == 0)
  {
    std::string elementType = typeName.substr(0, typeName.size() - 2);
    AddUniqueScanCandidate(candidates, elementType);
  }

  size_t genericOpen = typeName.find('<');
  if (genericOpen != std::string::npos)
  {
    std::string genericBase = TrimCopy(typeName.substr(0, genericOpen));
    int arity = CountGenericTopLevelArgs(typeName);
    AddUniqueScanCandidate(candidates, genericBase);
    if (arity > 0)
      AddUniqueScanCandidate(candidates, genericBase + "`" + std::to_string(arity));

    size_t lastDot = genericBase.rfind('.');
    if (lastDot != std::string::npos && lastDot + 1 < genericBase.size())
    {
      std::string shortBase = genericBase.substr(lastDot + 1);
      AddUniqueScanCandidate(candidates, shortBase);
      if (arity > 0)
        AddUniqueScanCandidate(candidates, shortBase + "`" + std::to_string(arity));
    }
  }

  return candidates;
}

static std::vector<std::string> SplitTopLevelGenericArgs(const std::string &typeName)
{
  std::vector<std::string> args;
  size_t open = typeName.find('<');
  size_t close = typeName.rfind('>');
  if (open == std::string::npos || close == std::string::npos || close <= open + 1)
    return args;

  int depth = 0;
  size_t start = open + 1;
  for (size_t i = open + 1; i < close; ++i)
  {
    char c = typeName[i];
    if (c == '<')
      ++depth;
    else if (c == '>')
    {
      if (depth > 0)
        --depth;
    }
    else if (c == ',' && depth == 0)
    {
      AddUniqueScanCandidate(args, typeName.substr(start, i - start));
      start = i + 1;
    }
  }
  AddUniqueScanCandidate(args, typeName.substr(start, close - start));
  return args;
}

static Il2CppClass *FindClassByScanCandidates(const std::string &typeName, std::string *resolvedName)
{
  std::vector<std::string> candidates = BuildScanClassCandidates(typeName);
  for (const std::string &candidate : candidates)
  {
    Il2CppClass *klass = Il2cpp::FindClass(candidate.c_str());
    if (klass)
    {
      if (resolvedName)
        *resolvedName = candidate;
      return klass;
    }
  }
  return nullptr;
}

static Il2CppClass *FindInflatedGenericClassForScan(const std::string &typeName, std::string *resolvedName)
{
  std::string clean = TrimCopy(typeName);
  size_t genericOpen = clean.find('<');
  if (genericOpen == std::string::npos || clean.rfind('>') == std::string::npos)
    return nullptr;

  std::string genericBase = TrimCopy(clean.substr(0, genericOpen));
  std::vector<std::string> argNames = SplitTopLevelGenericArgs(clean);
  if (genericBase.empty() || argNames.empty())
    return nullptr;

  std::string baseResolved;
  Il2CppClass *genericBaseClass = FindClassByScanCandidates(genericBase + "`" + std::to_string(argNames.size()), &baseResolved);
  if (!genericBaseClass)
    genericBaseClass = FindClassByScanCandidates(genericBase, &baseResolved);
  if (!genericBaseClass)
    return nullptr;

  std::vector<Il2CppClass *> argClasses;
  std::vector<std::string> argResolvedNames;
  for (const std::string &argName : argNames)
  {
    std::string argResolved;
    Il2CppClass *argClass = FindInflatedGenericClassForScan(argName, &argResolved);
    if (!argClass)
      argClass = FindClassByScanCandidates(argName, &argResolved);
    if (!argClass)
      return nullptr;

    argClasses.push_back(argClass);
    argResolvedNames.push_back(argResolved.empty() ? argName : argResolved);
  }

  Il2CppClass *inflated = nullptr;
  switch (argClasses.size())
  {
  case 1:
    inflated = genericBaseClass->inflate({argClasses[0]});
    break;
  case 2:
    inflated = genericBaseClass->inflate({argClasses[0], argClasses[1]});
    break;
  case 3:
    inflated = genericBaseClass->inflate({argClasses[0], argClasses[1], argClasses[2]});
    break;
  case 4:
    inflated = genericBaseClass->inflate({argClasses[0], argClasses[1], argClasses[2], argClasses[3]});
    break;
  default:
    LOGE("ObjectScan: generic arity %zu unsupported for %s", argClasses.size(), clean.c_str());
    return nullptr;
  }

  if (inflated && resolvedName)
  {
    *resolvedName = baseResolved + "[";
    for (size_t i = 0; i < argResolvedNames.size(); ++i)
    {
      if (i > 0)
        *resolvedName += ", ";
      *resolvedName += argResolvedNames[i];
    }
    *resolvedName += "]";
  }

  return inflated;
}

static Il2CppClass *FindClassForObjectScan(const std::string &typeName, std::string *resolvedName)
{
  Il2CppClass *inflated = FindInflatedGenericClassForScan(typeName, resolvedName);
  if (inflated)
    return inflated;

  Il2CppClass *klass = FindClassByScanCandidates(typeName, resolvedName);
  if (klass)
    return klass;

  if (resolvedName)
    resolvedName->clear();
  return nullptr;
}

static void OpenCallerObjectPicker(const std::string &className,
                                   const std::string &title,
                                   const std::string &slotKey)
{
  g_CallerObjectPickerClassName = className;
  g_CallerObjectPickerTitle = title;
  g_CallerObjectPickerSlotKey = slotKey;
  g_CallerObjectPickerOpen = true;
  g_FitCallerObjectPickerNextOpen = true;
  g_ScanClassName = className;
  g_ScannedObjects.clear();
  g_ScanRequested = true;
}

static std::string FormatObjectAddress(uintptr_t address)
{
  if (!address)
    return "nil";
  char buf[32];
  snprintf(buf, sizeof(buf), "0x%llX", (unsigned long long)address);
  return buf;
}

static std::string ObjectExprFromAddress(uintptr_t address)
{
  return address ? ("Object.fromAddress(" + FormatObjectAddress(address) + ")") : "nil";
}

static std::string BuildSetFieldAllScript(const std::string &className,
                                          const std::string &fieldName,
                                          const std::string &setterName,
                                          const std::string &valueExpr)
{
  return "local c = Class.fromName(" + LuaQuote(className) + ")\n"
                                                             "if not c then return end\n"
                                                             "local done = 0\n"
                                                             "for _, obj in ipairs(c:findObjects() or {}) do\n"
                                                             "    pcall(function()\n"
                                                             "        obj:" +
         setterName + "(" + LuaQuote(fieldName) + ", " + valueExpr + ")\n"
                                                                     "        done = done + 1\n"
                                                                     "    end)\n"
                                                                     "end\n"
                                                                     "print(\"SetFieldAll " +
         fieldName + " done=\" .. tostring(done))";
}

static void PublishSetFieldAllScript(const std::string &script)
{
  LuaConsole_SetScriptText(script.c_str());
  CopyToClipboard(script.c_str());
  g_ActiveTabIndex = 1;
}

static std::string GenerateSetFieldScript(void *tabRootId,
                                          const std::string &className,
                                          const std::string &fieldName,
                                          const std::string &setterName,
                                          const std::string &valueExpr)
{
  if (tabRootId)
  {
    auto &history = g_TabHistories[tabRootId];
    if (history.size() > 1)
    {
      const char *rNameRaw = Il2cpp::GetClassName(history[0].klass);
      const char *rNsRaw = Il2cpp::GetClassNamespace(history[0].klass);
      std::string rootClassName = (rNsRaw && rNsRaw[0])
                                      ? (std::string(rNsRaw) + "." + (rNameRaw ? rNameRaw : ""))
                                      : (rNameRaw ? rNameRaw : "");
      std::string script;
      script += "local c = Class.fromName(" + LuaQuote(rootClassName) + ")\n";
      script += "if not c then return end\n";
      script += "local done = 0\n";
      script += "for _, obj in ipairs(c:findObjects() or {}) do\n";
      script += "    pcall(function()\n";

      std::string currentVar = "obj";
      int varCount = 1;
      int indentLevel = 2;

      auto getIndent = [](int level)
      {
        return std::string(level * 4, ' ');
      };

      for (size_t i = 1; i < history.size(); ++i)
      {
        std::string nextVar = "v" + std::to_string(varCount++);
        std::string label = history[i].label;

        script += getIndent(indentLevel);
        if (label.size() > 2 && label.front() == '[' && label.back() == ']')
        {
          std::string idxStr = label.substr(1, label.size() - 2);
          int idx = std::stoi(idxStr);
          size_t offset = 32 + (size_t)idx * 8;

          std::string parentKlassName = CleanTypeName(Il2cpp::GetClassName(history[i - 1].klass));
          if (parentKlassName.find("List`1") != std::string::npos || parentKlassName.find("List<") != std::string::npos)
          {
            script += "local items = " + currentVar + ":getFieldObject(\"_items\")\n";
            script += getIndent(indentLevel);
            script += "local " + nextVar + " = items and items:getObjectAt(" + std::to_string(offset) + ")\n";
          }
          else
          {
            script += "local " + nextVar + " = " + currentVar + ":getObjectAt(" + std::to_string(offset) + ")\n";
          }
        }
        else
        {
          script += "local " + nextVar + " = " + currentVar + ":getFieldObject(" + LuaQuote(label) + ")\n";
        }

        script += getIndent(indentLevel);
        script += "if " + nextVar + " then\n";
        currentVar = nextVar;
        indentLevel++;
      }

      script += getIndent(indentLevel);
      script += currentVar + ":" + setterName + "(" + LuaQuote(fieldName) + ", " + valueExpr + ")\n";
      script += getIndent(indentLevel);
      script += "done = done + 1\n";

      for (int level = indentLevel - 1; level >= 2; --level)
      {
        script += getIndent(level);
        script += "end\n";
      }

      script += "    end)\n";
      script += "end\n";
      script += "print(\"SetFieldAll " + fieldName + " done=\" .. tostring(done))";
      return script;
    }
  }

  return BuildSetFieldAllScript(className, fieldName, setterName, valueExpr);
}

static std::string GenerateBatchScript()
{
  std::string fullScript;
  
  std::vector<std::pair<std::string, std::vector<QueuedSetField>>> groups;
  for (const auto &q : g_BatchFieldQueue)
  {
      std::string key = GetPathKey(q);
      bool found = false;
      for (auto &group : groups)
      {
          if (group.first == key)
          {
              group.second.push_back(q);
              found = true;
              break;
          }
      }
      if (!found)
      {
          groups.push_back({key, {q}});
      }
  }

  for (const auto &group : groups)
  {
      const auto &first = group.second[0];
      if (first.historyPath.size() > 1)
      {
          fullScript += "-- Batch set for path: " + group.first + "\n";
          fullScript += "local c = Class.fromName(" + LuaQuote(first.historyPath[0].className) + ")\n";
          fullScript += "if c then\n";
          fullScript += "    local done = 0\n";
          fullScript += "    for _, obj in ipairs(c:findObjects() or {}) do\n";
          fullScript += "        pcall(function()\n";

          std::string currentVar = "obj";
          int varCount = 1;
          int indentLevel = 3;

          auto getIndent = [](int level)
          {
            return std::string(level * 4, ' ');
          };

          for (size_t i = 1; i < first.historyPath.size(); ++i)
          {
            std::string nextVar = "v" + std::to_string(varCount++);
            std::string label = first.historyPath[i].label;

            fullScript += getIndent(indentLevel);
            if (label.size() > 2 && label.front() == '[' && label.back() == ']')
            {
              std::string idxStr = label.substr(1, label.size() - 2);
              int idx = std::stoi(idxStr);
              size_t offset = 32 + (size_t)idx * 8;

              std::string parentKlassName = CleanTypeName(first.historyPath[i - 1].className.c_str());
              if (parentKlassName.find("List`1") != std::string::npos || parentKlassName.find("List<") != std::string::npos)
              {
                fullScript += "local items = " + currentVar + ":getFieldObject(\"_items\")\n";
                fullScript += getIndent(indentLevel);
                fullScript += "local " + nextVar + " = items and items:getObjectAt(" + std::to_string(offset) + ")\n";
              }
              else
              {
                fullScript += "local " + nextVar + " = " + currentVar + ":getObjectAt(" + std::to_string(offset) + ")\n";
              }
            }
            else
            {
              fullScript += "local " + nextVar + " = " + currentVar + ":getFieldObject(" + LuaQuote(label) + ")\n";
            }

            fullScript += getIndent(indentLevel);
            fullScript += "if " + nextVar + " then\n";
            currentVar = nextVar;
            indentLevel++;
          }

          // Now set all fields in this group on currentVar:
          for (const auto &q : group.second)
          {
            fullScript += getIndent(indentLevel);
            fullScript += currentVar + ":" + q.setterName + "(" + LuaQuote(q.fieldName) + ", " + q.valueExpr + ")\n";
          }

          fullScript += getIndent(indentLevel);
          fullScript += "done = done + 1\n";

          for (int level = indentLevel - 1; level >= 3; --level)
          {
            fullScript += getIndent(level);
            fullScript += "end\n";
          }

          fullScript += "        end)\n";
          fullScript += "    end\n";
          fullScript += "    print(\"Batch " + first.className + " done=\" .. tostring(done))\n";
          fullScript += "end\n";
      }
      else
      {
          fullScript += "-- Batch set for: " + first.className + "\n";
          fullScript += "local c = Class.fromName(" + LuaQuote(first.className) + ")\n";
          fullScript += "if c then\n";
          fullScript += "    local done = 0\n";
          fullScript += "    for _, obj in ipairs(c:findObjects() or {}) do\n";
          fullScript += "        pcall(function()\n";

          for (const auto &q : group.second)
          {
            fullScript += "            obj:" + q.setterName + "(" + LuaQuote(q.fieldName) + ", " + q.valueExpr + ")\n";
          }

          fullScript += "            done = done + 1\n";
          fullScript += "        end)\n";
          fullScript += "    end\n";
          fullScript += "    print(\"Batch " + first.className + " done=\" .. tostring(done))\n";
          fullScript += "end\n";
      }
  }
  return fullScript;
}

static void HandleCopyField(void *tabRootId,
                            const std::string &className,
                            const std::string &fieldName,
                            const std::string &setterName,
                            const std::string &valueExpr,
                            bool isAllAction = false)
{
  LOGI("HandleCopyField called: field=%s, class=%s, setter=%s, value=%s, batchMode=%d, isAll=%d", 
       fieldName.c_str(), className.c_str(), setterName.c_str(), valueExpr.c_str(), g_BatchScriptMode, (int)isAllAction);
  if (isAllAction)
  {
    std::string script = GenerateSetFieldScript(tabRootId, className, fieldName, setterName, valueExpr);
    PublishSetFieldAllScript(script);
    LuaConsole_RequestExecute(script.c_str());
  }
  else if (g_BatchScriptMode)
  {
    QueuedSetField q;
    q.tabRootId = tabRootId;
    q.className = className;
    q.fieldName = fieldName;
    q.setterName = setterName;
    q.valueExpr = valueExpr;

    if (tabRootId)
    {
      auto &history = g_TabHistories[tabRootId];
      for (const auto &h : history)
      {
        PathNode node;
        const char *rNameRaw = Il2cpp::GetClassName(h.klass);
        const char *rNsRaw = Il2cpp::GetClassNamespace(h.klass);
        node.className = (rNsRaw && rNsRaw[0])
                             ? (std::string(rNsRaw) + "." + (rNameRaw ? rNameRaw : ""))
                             : (rNameRaw ? rNameRaw : "");
        node.label = h.label;
        q.historyPath.push_back(node);
      }
    }
    g_BatchFieldQueue.push_back(q);
  }
  else
  {
    PublishSetFieldAllScript(GenerateSetFieldScript(tabRootId, className, fieldName, setterName, valueExpr));
  }
}

static std::string BuildCallResultCaptureSnippet(const MethodEntry &method)
{
  if (LowerCopy(method.returnType) == "void" || method.returnType.find("System.Void") != std::string::npos)
  {
    return "        results[#results + 1] = \"CALL_RESULT:void:null\"\n";
  }

  std::string retType = method.returnType.empty() ? "object" : method.returnType;
  std::string typeExpr = LuaQuote(retType);
  std::string prefix = "CALL_RESULT:" + retType + ":";

  if (IsTaskTypeName(method.returnType))
  {
    return "        results[#results + 1] = \"CALL_RESULT:" + retType + ":\" .. __call_format_value(ret, " + typeExpr + ")\n"
                                                                                                                        "        local okCompleted, completed = pcall(function() return ret:get_IsCompleted() end)\n"
                                                                                                                        "        if okCompleted then results[#results + 1] = \"CALL_RESULT:Task.IsCompleted:\" .. tostring(completed) end\n"
                                                                                                                        "        local okFaulted, faulted = pcall(function() return ret:get_IsFaulted() end)\n"
                                                                                                                        "        if okFaulted then results[#results + 1] = \"CALL_RESULT:Task.IsFaulted:\" .. tostring(faulted) end\n"
                                                                                                                        "        local okCanceled, canceled = pcall(function() return ret:get_IsCanceled() end)\n"
                                                                                                                        "        if okCanceled then results[#results + 1] = \"CALL_RESULT:Task.IsCanceled:\" .. tostring(canceled) end\n"
                                                                                                                        "        if okCompleted and completed and not (okFaulted and faulted) and not (okCanceled and canceled) then\n"
                                                                                                                        "            local okResult, result = pcall(function() return ret:get_Result() end)\n"
                                                                                                                        "            if okResult then results[#results + 1] = \"CALL_RESULT:Task.Result:\" .. __call_format_value(result, \"Task.Result\") end\n"
                                                                                                                        "        else\n"
                                                                                                                        "            results[#results + 1] = \"CALL_RESULT:Task.Result:pending\"\n"
                                                                                                                        "        end\n";
  }

  return "        results[#results + 1] = " + LuaQuote(prefix) + " .. __call_format_value(ret, " + typeExpr + ")\n";
}

static std::string BuildCallFormatLua()
{
  return "local function __call_format_value(v, declaredType)\n"
         "    if v == nil then return \"null\" end\n"
         "    local tv = type(v)\n"
         "    if tv == \"number\" or tv == \"boolean\" or tv == \"string\" then return tostring(v) end\n"
         "    local s = tostring(v)\n"
         "    local okClass, className = pcall(function()\n"
         "        if v.getClassName then return v:getClassName() end\n"
         "        if v.getClass then return tostring(v:getClass()) end\n"
         "        return nil\n"
         "    end)\n"
         "    if okClass and className and tostring(className) ~= \"nil\" then\n"
         "        return tostring(className) .. \" \" .. s\n"
         "    end\n"
         "    return s\n"
         "end\n";
}

static std::string BuildCallScript(const std::string &className,
                                   const MethodEntry &method,
                                   const std::vector<std::string> &argExprs,
                                   uintptr_t selectedObject,
                                   int callCount)
{
  callCount = std::max(1, std::min(callCount, 100));
  std::string args;
  for (size_t i = 0; i < argExprs.size(); ++i)
  {
    if (i > 0)
      args += ", ";
    args += argExprs[i].empty() ? "nil" : argExprs[i];
  }

  if (selectedObject)
  {
    char addrBuf[32];
    snprintf(addrBuf, sizeof(addrBuf), "0x%llX", (unsigned long long)selectedObject);
    return BuildCallFormatLua() +
           "local obj = Object.fromAddress(" + std::string(addrBuf) + ")\n"
                                                                      "if not obj then return end\n"
                                                                      "local callCount = " +
           std::to_string(callCount) + "\n"
                                       "local done = 0\n"
                                       "local results = {}\n"
                                       "local errors = {}\n"
                                       "for callIndex = 1, callCount do\n"
                                       "    local ok, ret = pcall(function()\n"
                                       "        return obj:" +
           method.name + "(" + args + ")\n"
                                      "    end)\n"
                                      "    if ok then\n"
                                      "        done = done + 1\n"
                                      "        if true then\n" +
           BuildCallResultCaptureSnippet(method) +
           "        end\n"
           "    else\n"
           "        errors[#errors + 1] = tostring(ret)\n"
           "    end\n"
           "    if callIndex < callCount then sleepMs(2000) end\n"
           "end\n"
           "print(\"Call " +
           method.name + " done=\" .. tostring(done))\n"
                         "if #results > 0 then\n"
                         "    print(\"Call Results:\")\n"
                         "    for i, v in ipairs(results) do print(v) end\n"
                         "end\n"
                         "if #errors > 0 then\n"
                         "    print(\"Call Errors:\")\n"
                         "    for i, v in ipairs(errors) do print(v) end\n"
                         "end";
  }

  return BuildCallFormatLua() +
         "local c = Class.fromName(" + LuaQuote(className) + ")\n"
                                                             "if not c then return end\n"
                                                             "local callCount = " +
         std::to_string(callCount) + "\n"
                                     "local done = 0\n"
                                     "local results = {}\n"
                                     "local errors = {}\n"
                                     "for _, obj in ipairs(c:findObjects() or {}) do\n"
                                     "    for callIndex = 1, callCount do\n"
                                     "        local ok, ret = pcall(function()\n"
                                     "            return obj:" +
         method.name + "(" + args + ")\n"
                                    "        end)\n"
                                    "        if ok then\n"
                                    "            done = done + 1\n"
                                    "            if true then\n" +
         BuildCallResultCaptureSnippet(method) +
         "            end\n"
         "        else\n"
         "            errors[#errors + 1] = tostring(ret)\n"
         "        end\n"
         "        sleepMs(500)\n"
         "    end\n"
         "end\n"
         "print(\"Call " +
         method.name + " done=\" .. tostring(done))\n"
                       "if #results > 0 then\n"
                       "    print(\"Call Results:\")\n"
                       "    for i, v in ipairs(results) do print(v) end\n"
                       "end\n"
                       "if #errors > 0 then\n"
                       "    print(\"Call Errors:\")\n"
                       "    for i, v in ipairs(errors) do print(v) end\n"
                       "end";
}

static std::vector<std::string> SplitLines(const char *text)
{
  std::vector<std::string> lines;
  if (!text)
    return lines;

  std::string current;
  for (const char *p = text; *p; ++p)
  {
    if (*p == '\n')
    {
      if (!current.empty() && current.back() == '\r')
        current.pop_back();
      lines.push_back(current);
      current.clear();
    }
    else
    {
      current.push_back(*p);
    }
  }
  if (!current.empty())
    lines.push_back(current);
  return lines;
}

static std::vector<std::string> ExtractCallResultValues(const char *output)
{
  std::vector<std::string> values;
  std::vector<std::string> lines = SplitLines(output);
  bool inResults = false;
  for (const auto &line : lines)
  {
    if (line == "Call Results:")
    {
      inResults = true;
      continue;
    }
    if (!inResults)
      continue;
    if (line.empty())
      continue;
    if (line.rfind("Call ", 0) == 0)
      break;
    const std::string marker = "CALL_RESULT:";
    if (line.rfind(marker, 0) == 0)
    {
      size_t typeEnd = line.find(':', marker.size());
      if (typeEnd != std::string::npos)
        values.push_back(line.substr(typeEnd + 1));
      else
        values.push_back(line.substr(marker.size()));
    }
    else
    {
      values.push_back(line);
    }
  }
  return values;
}

static void RenderCopyableResultValue(const std::string &value, const std::string &type, const char *id)
{
  ImGui::PushID(id);
  ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.25f, 0.43f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.28f, 0.35f, 0.60f, 1.00f));
  ImGui::Button(value.c_str());
  ImGui::PopStyleColor(2);

  RenderCopyPopupForLastItem("result_value", "Copy Result", {{"Value", value}, {"Type", type}, {"Address", value}});

  ImGui::SameLine();
  if (ImGui::SmallButton("Copy"))
  {
    CopyToClipboard(value.c_str());
  }
  ImGui::PopID();
}

static void CapturePendingCallerResult()
{
  if (g_PendingCallerResultKey.empty())
    return;

  const char *outputText = LuaConsole_GetOutputText();
  if (!outputText || !outputText[0])
    return;

  std::string output = outputText;
  if (output.find("Queued on Unity main thread") != std::string::npos)
    return;

  std::vector<std::string> values = ExtractCallResultValues(outputText);
  if (!values.empty())
  {
    auto &stored = g_CallerResultValues[g_PendingCallerResultKey];
    for (const auto &value : values)
    {
      stored.push_back(value);
    }
    while (stored.size() > 5)
    {
      stored.erase(stored.begin());
    }
    g_CallerResultRaw.erase(g_PendingCallerResultKey);
  }
  else
  {
    g_CallerResultRaw[g_PendingCallerResultKey] = output;
  }

  g_LastCallerCapturedOutput = output;
  g_PendingCallerResultKey.clear();
}

static const PatchEntry *FindRedirectPatchForMethod(const std::vector<PatchEntry> &patches,
                                                    const std::string &className,
                                                    const std::string &methodName)
{
  auto shortClassName = [](const std::string &name)
  {
    size_t dot = name.find_last_of('.');
    return dot == std::string::npos ? name : name.substr(dot + 1);
  };
  for (const auto &patch : patches)
  {
    bool isMethodPatch = patch.type == "Direct" ||
                         patch.type == "Redirect0" ||
                         patch.type.rfind("Return", 0) == 0;
    if (isMethodPatch &&
        (patch.sourceClass == className || patch.sourceClass == shortClassName(className)) &&
        patch.sourceMethod == methodName)
    {
      return &patch;
    }
  }
  return nullptr;
}

static const ClassEntry *FindCachedClass(const std::string &fullName)
{
  for (const auto &img : g_MetadataCache)
  {
    for (const auto &cls : img.classes)
    {
      std::string clsFullName = cls.ns.empty() ? cls.name : (cls.ns + "." + cls.name);
      if (clsFullName == fullName)
        return &cls;
    }
  }
  return nullptr;
}

static void RenderRedirectChooserWindow()
{
  if (!g_ShowRedirectChooserWindow)
  {
    s_redirectRect.active = false;
    return;
  }

  static char targetClassBuf[256] = "";
  static char methodFilterBuf[128] = "";
  static bool lastOpen = false;

  if (g_FitRedirectChooserWindowNextOpen)
  {
    ImGui::SetNextWindowSize(ImVec2(360 * g_FontScale, 360 * g_FontScale), ImGuiCond_Always);
    g_FitRedirectChooserWindowNextOpen = false;
  }
  else
  {
    ImGui::SetNextWindowSize(ImVec2(360 * g_FontScale, 360 * g_FontScale), ImGuiCond_FirstUseEver);
  }
  ImGui::SetNextWindowSizeConstraints(ImVec2(300 * g_FontScale, 220 * g_FontScale), ImVec2(520 * g_FontScale, 560 * g_FontScale));
  ImGui::SetNextWindowPos(ImVec2(100 * g_FontScale, 100 * g_FontScale), ImGuiCond_FirstUseEver);

  if (ImGui::Begin("Redirect Method##floatWin", &g_ShowRedirectChooserWindow))
  {
    // Update touch-region so the overlay captures touches over this window
    ImVec2 fwPos = ImGui::GetWindowPos();
    ImVec2 fwSize = ImGui::GetWindowSize();
    s_redirectRect = {true, fwPos.x, fwPos.y, fwSize.x, fwSize.y};

    if (!lastOpen)
    {
      snprintf(targetClassBuf, sizeof(targetClassBuf), "%s", g_RedirectSourceClass.c_str());
      methodFilterBuf[0] = '\0';
    }
    lastOpen = true;

    ImGui::TextDisabled("%s.%s", g_RedirectSourceClass.c_str(), g_RedirectSourceMethod.name.c_str());
    ImGui::Separator();

    float availW = ImGui::GetContentRegionAvail().x;
    ImGui::SetNextItemWidth(std::max(180.0f, availW - ImGui::CalcTextSize("Class").x - ImGui::GetStyle().ItemSpacing.x));
    if (ImGui::BeginCombo("Class", targetClassBuf[0] ? targetClassBuf : "Select class"))
    {
      for (const auto &img : g_MetadataCache)
      {
        for (const auto &cls : img.classes)
        {
          std::string clsFullName = cls.ns.empty() ? cls.name : (cls.ns + "." + cls.name);
          bool selected = (clsFullName == targetClassBuf);
          if (ImGui::Selectable(clsFullName.c_str(), selected))
          {
            snprintf(targetClassBuf, sizeof(targetClassBuf), "%s", clsFullName.c_str());
            methodFilterBuf[0] = '\0';
          }
        }
      }
      ImGui::EndCombo();
    }

    ImGui::SetNextItemWidth(std::max(180.0f, availW - ImGui::CalcTextSize("Filter").x - ImGui::GetStyle().ItemSpacing.x));
    ImGui::InputText("Filter", methodFilterBuf, sizeof(methodFilterBuf));
    ImGui::Separator();

    const ClassEntry *targetClass = FindCachedClass(targetClassBuf);
    if (targetClass)
    {
      std::string filter = methodFilterBuf;
      std::transform(filter.begin(), filter.end(), filter.begin(), ::tolower);

      ImGui::BeginChild("##RedirectMethodList", ImVec2(0.0f, 0.0f), true, ImGuiWindowFlags_HorizontalScrollbar);
      for (const auto &targetMethod : targetClass->methods)
      {
        std::string label = targetMethod.signature;
        std::string lowerLabel = label;
        std::transform(lowerLabel.begin(), lowerLabel.end(), lowerLabel.begin(), ::tolower);
        if (!filter.empty() && lowerLabel.find(filter) == std::string::npos)
          continue;

        if (ImGui::Selectable(label.c_str(), false))
        {
          std::string script = BuildRedirectScript(g_RedirectSourceClass, g_RedirectSourceMethod, targetClassBuf, targetMethod);
          LuaConsole_SetScriptText(script.c_str());
          LuaConsole_RequestExecute(script.c_str());
          g_ActiveTabIndex = 1;
          g_ShowRedirectChooserWindow = false;
        }
      }
      ImGui::EndChild();
    }
    else
    {
      ImGui::TextDisabled("Class not found in cache.");
    }
  }
  else
  {
    s_redirectRect.active = false;
    lastOpen = false;
  }
  ImGui::End();
}

static void RenderInstancesWindow()
{
  if (!g_ShowInstancesWindow)
  {
    s_instancesRect.active = false;
    return;
  }

  if (g_FitInstancesWindowNextOpen)
  {
    ImGui::SetNextWindowSize(ImVec2(360 * g_FontScale, 360 * g_FontScale), ImGuiCond_Always);
    g_FitInstancesWindowNextOpen = false;
  }
  else
  {
    ImGui::SetNextWindowSize(ImVec2(360 * g_FontScale, 360 * g_FontScale), ImGuiCond_FirstUseEver);
  }
  ImGui::SetNextWindowSizeConstraints(ImVec2(300 * g_FontScale, 220 * g_FontScale), ImVec2(520 * g_FontScale, 560 * g_FontScale));
  ImGui::SetNextWindowPos(ImVec2(100 * g_FontScale, 100 * g_FontScale), ImGuiCond_FirstUseEver);

  if (ImGui::Begin("Inspector##floatWin", &g_ShowInstancesWindow))
  {
    // Update touch-region so the overlay captures touches over this window
    ImVec2 fwPos = ImGui::GetWindowPos();
    ImVec2 fwSize = ImGui::GetWindowSize();
    s_instancesRect = {true, fwPos.x, fwPos.y, fwSize.x, fwSize.y};

    float wAvail = ImGui::GetContentRegionAvail().x;
    const float fPad = ImGui::GetStyle().FramePadding.x * 2.0f;
    const float fSp = ImGui::GetStyle().ItemSpacing.x;
    float findBtnW = ImGui::CalcTextSize("Find Objects").x + fPad + 8.0f;
    float filterW = ImGui::CalcTextSize("Filter####").x + fPad + 8.0f;

    if (ImGui::Button("Find Objects", ImVec2(findBtnW, 0)))
    {
      g_ScannedObjects.clear();
      if (!g_ScanClassName.empty())
        g_ScanRequested = true;
    }

    ImGui::SameLine(wAvail - filterW + fSp);
    ImGui::SetNextItemWidth(filterW);
    ImGui::InputText("##ObjFilter", g_FindObjectsFilter, sizeof(g_FindObjectsFilter));
    if (ImGui::IsItemHovered() && g_FindObjectsFilter[0] == '\0')
      ImGui::SetTooltip("Filter");

    ImGui::Separator();
    float manualBtnW = ImGui::CalcTextSize("Manual Input").x + fPad + 8.0f;
    float manualInputW = wAvail - manualBtnW - fSp - 4.0f;
    ImGui::SetNextItemWidth(manualInputW);
    ImGui::InputText("##manualAddr", g_ManualInputAddress, sizeof(g_ManualInputAddress));
    ImGui::SameLine();
    if (ImGui::Button("Manual Input", ImVec2(manualBtnW, 0)))
    {
      unsigned long long addr = 0;
      if (sscanf(g_ManualInputAddress, "%llX", &addr) == 1 ||
          sscanf(g_ManualInputAddress, "0x%llX", &addr) == 1)
      {
        if (addr != 0)
        {
          OpenInspectedObjectDump(reinterpret_cast<void *>(addr));
        }
      }
    }
    ImGui::Separator();

    Il2CppClass *instKlass = Il2cpp::FindClass(g_ScanClassName.c_str());
    if (instKlass)
    {
      if (ImGui::CollapsingHeader("Create New Instance", ImGuiTreeNodeFlags_None))
      {
        void *iter = nullptr;
        int ctorIndex = 0;
        while (auto m = Il2cpp::GetClassMethods(instKlass, &iter))
        {
          if (strcmp(Il2cpp::GetMethodName(m), ".ctor") != 0)
            continue;

          ctorIndex++;
          ImGui::PushID(ctorIndex);

          std::string ctorLabel = "new (";
          int paramCount = Il2cpp::GetMethodParamCount(m);
          for (int i = 0; i < paramCount; ++i)
          {
            Il2CppType *pType = Il2cpp::GetMethodParam(m, i);
            const char *pTypeName = Il2cpp::GetTypeName(pType);
            if (i > 0)
              ctorLabel += ", ";
            ctorLabel += pTypeName ? pTypeName : "object";
          }
          ctorLabel += ")";

          if (ImGui::TreeNode(ctorLabel.c_str()))
          {
            // Inputs for constructor parameters
            static char ctorArgs[16][128] = {};
            static int ctorInts[16] = {};
            static float ctorFloats[16] = {};
            static double ctorDoubles[16] = {};
            static bool ctorBools[16] = {};

            for (int p = 0; p < paramCount; ++p)
            {
              Il2CppType *param = Il2cpp::GetMethodParam(m, p);
              std::string label = std::string(Il2cpp::GetTypeName(param)) + " arg" + std::to_string(p);
              ImGui::PushID(p);

              if (IsBoolTypeName(Il2cpp::GetTypeName(param)))
              {
                int choice = ctorBools[p] ? 1 : 0;
                const char *items[] = {"false", "true"};
                ImGui::SetNextItemWidth(120.0f * g_FontScale);
                if (ImGui::Combo(label.c_str(), &choice, items, 2))
                  ctorBools[p] = choice != 0;
              }
              else if (IsIntTypeName(Il2cpp::GetTypeName(param)) || IsLongTypeName(Il2cpp::GetTypeName(param)))
              {
                ImGui::SetNextItemWidth(120.0f * g_FontScale);
                ImGui::InputInt(label.c_str(), &ctorInts[p]);
              }
              else if (IsFloatTypeName(Il2cpp::GetTypeName(param)))
              {
                ImGui::SetNextItemWidth(120.0f * g_FontScale);
                ImGui::InputFloat(label.c_str(), &ctorFloats[p]);
              }
              else if (IsDoubleTypeName(Il2cpp::GetTypeName(param)))
              {
                ImGui::SetNextItemWidth(120.0f * g_FontScale);
                ImGui::InputDouble(label.c_str(), &ctorDoubles[p]);
              }
              else if (IsStringTypeName(Il2cpp::GetTypeName(param)))
              {
                ImGui::SetNextItemWidth(180.0f * g_FontScale);
                ImGui::InputText(label.c_str(), ctorArgs[p], sizeof(ctorArgs[p]));
              }
              else
              {
                // Object parameter selection
                std::string paramSlotKey = "ctor_inst_" + std::to_string(ctorIndex) + "_arg_" + std::to_string(p);
                uintptr_t selectedParamObj = g_CallerObjectSelections[paramSlotKey];
                std::string argComboPreview = selectedParamObj ? FormatObjectAddress(selectedParamObj) : "nil";
                ImGui::SetNextItemWidth(180.0f * g_FontScale);
                if (ImGui::BeginCombo(label.c_str(), argComboPreview.c_str()))
                {
                  OpenCallerObjectPicker(Il2cpp::GetTypeName(param), label, paramSlotKey);
                  ImGui::EndCombo();
                }
              }
              ImGui::PopID();
            }

            if (ImGui::Button("Create Instance"))
            {
              // Build Lua Script to call: Class.fromName("ClassName"):new(...)
              std::string callScript = "local klass = Class.fromName(\"" + g_ScanClassName + "\")\n";
              callScript += "local obj = klass:new(";
              for (int p = 0; p < paramCount; ++p)
              {
                if (p > 0)
                  callScript += ", ";
                Il2CppType *param = Il2cpp::GetMethodParam(m, p);
                if (IsBoolTypeName(Il2cpp::GetTypeName(param)))
                {
                  callScript += ctorBools[p] ? "true" : "false";
                }
                else if (IsIntTypeName(Il2cpp::GetTypeName(param)) || IsLongTypeName(Il2cpp::GetTypeName(param)))
                {
                  callScript += std::to_string(ctorInts[p]);
                }
                else if (IsFloatTypeName(Il2cpp::GetTypeName(param)))
                {
                  callScript += std::to_string(ctorFloats[p]);
                }
                else if (IsDoubleTypeName(Il2cpp::GetTypeName(param)))
                {
                  callScript += std::to_string(ctorDoubles[p]);
                }
                else if (IsStringTypeName(Il2cpp::GetTypeName(param)))
                {
                  callScript += LuaQuote(ctorArgs[p]);
                }
                else
                {
                  std::string paramSlotKey = "ctor_inst_" + std::to_string(ctorIndex) + "_arg_" + std::to_string(p);
                  uintptr_t selectedParamObj = g_CallerObjectSelections[paramSlotKey];
                  callScript += selectedParamObj ? ObjectExprFromAddress(selectedParamObj) : "nil";
                }
              }
              callScript += ")\n";
              callScript += "if obj then\n";
              callScript += "  registerCreatedObject(obj)\n";
              callScript += "  print(\"Created instance of \" .. \"" + g_ScanClassName + "\")\n";
              callScript += "else\n";
              callScript += "  print(\"Failed to create instance\")\n";
              callScript += "end\n";

              LuaConsole_RequestExecute(callScript.c_str());
            }

            ImGui::TreePop();
          }

          ImGui::PopID();
        }
      }
      ImGui::Separator();
    }

    if (g_ScanRequested || g_ScanInProgress)
    {
      ImGui::TextColored(ImVec4(1.0f, 0.8f, 0.2f, 1.0f), "Scanning heap on Main Thread...");
    }

    std::string scanShortName = g_ScanClassName;
    size_t dotPos = scanShortName.rfind('.');
    if (dotPos != std::string::npos)
      scanShortName = scanShortName.substr(dotPos + 1);

    char listHeader[128];
    snprintf(listHeader, sizeof(listHeader), "%s (%zu/0)",
             scanShortName.c_str(), g_ScannedGCHandles.size());
    ImGui::TextColored(ImVec4(0.9f, 0.9f, 0.9f, 1.0f), "%s", listHeader);

    ImGui::BeginChild("##InstList", ImVec2(0, 0), true);
    if (g_ScannedGCHandles.empty())
    {
      ImGui::TextDisabled("No instances found. Click Find Objects.");
    }
    else
    {
      for (uint32_t handle : g_ScannedGCHandles)
      {
        void *obj = Il2cpp::GCHandle_GetTarget(handle);
        if (!obj)
          continue;
        char objLabel[128];
        snprintf(objLabel, sizeof(objLabel), "%s 0x%llX",
                 scanShortName.c_str(), (unsigned long long)obj);

        if (g_FindObjectsFilter[0] != '\0')
        {
          std::string ls = objLabel, fs = g_FindObjectsFilter;
          std::transform(ls.begin(), ls.end(), ls.begin(), ::tolower);
          std::transform(fs.begin(), fs.end(), fs.begin(), ::tolower);
          if (ls.find(fs) == std::string::npos)
            continue;
        }

        bool isSel = (g_InspectedObject == obj);
        if (ImGui::Selectable(objLabel, isSel))
        {
          OpenInspectedObjectDump(obj);
        }
      }
    }
    ImGui::EndChild();
  }
  else
  {
    s_instancesRect.active = false;
  }
  ImGui::End();
}

struct MainLivenessContext
{
  std::vector<uint32_t> handles;
  int maxCount;
};

static void MainLivenessCallback(Il2CppObject **arr, int size, void *userdata)
{
  MainLivenessContext *ctx = (MainLivenessContext *)userdata;
  if (!ctx || !arr || size <= 0)
    return;

  for (int i = 0; i < size && (int)ctx->handles.size() < ctx->maxCount; ++i)
  {
    if (arr[i])
    {
      uint32_t handle = Il2cpp::GCHandle_New(arr[i], false);
      ctx->handles.push_back(handle);
    }
  }
}

static void *MainLivenessReallocate(void *ptr, size_t size, void *userdata)
{
  (void)userdata;
  return realloc(ptr, size);
}

static MethodInfo *FindZeroParamMethod(Il2CppClass *klass, const std::string &methodName)
{
  if (!klass)
    return nullptr;
  void *iter = nullptr;
  while (auto m = Il2cpp::GetClassMethods(klass, &iter))
  {
    const char *name = Il2cpp::GetMethodName(m);
    if (name && methodName == name && Il2cpp::GetMethodParamCount(m) == 0)
    {
      return m;
    }
  }
  return nullptr;
}

static MethodInfo *FindMethodForEntry(Il2CppClass *klass, const MethodEntry &methodEntry)
{
  if (!klass)
    return nullptr;

  uintptr_t il2cppBase = KittyMemory::getLibraryBaseMap("libil2cpp.so").startAddress;
  MethodInfo *nameFallback = nullptr;
  void *iter = nullptr;
  while (auto m = Il2cpp::GetClassMethods(klass, &iter))
  {
    const char *name = Il2cpp::GetMethodName(m);
    if (!name || methodEntry.name != name)
      continue;

    if (!nameFallback)
      nameFallback = m;

    if (methodEntry.rva != 0 && il2cppBase != 0 && m->methodPointer)
    {
      uint64_t rva = (uint64_t)reinterpret_cast<uintptr_t>(m->methodPointer) - il2cppBase;
      if (rva == methodEntry.rva)
        return m;
    }
  }
  return nameFallback;
}

static size_t AdjustValueTypeFieldOffset(size_t offset)
{
  size_t header = sizeof(Il2CppObject);
  return offset >= header ? offset - header : offset;
}

#include <unistd.h>
#include <fcntl.h>

static bool IsBadReadPtr(void *p)
{
  if (!p || (uintptr_t)p < 0x1000)
    return true;

  static int nullfd = -1;
  if (nullfd == -1)
  {
    nullfd = open("/dev/null", O_WRONLY);
  }
  if (nullfd < 0)
    return false;

  if (write(nullfd, p, 1) < 0)
  {
    return true;
  }
  return false;
}

static Il2CppClass* SafeGetClass(void *obj)
{
  if (!obj || (uintptr_t)obj < 0x1000)
    return nullptr;
  if (IsBadReadPtr(obj))
    return nullptr;
  if (((uintptr_t)obj & 7) != 0)
    return nullptr;

  void *klass = *reinterpret_cast<void **>(obj);
  if (!klass || IsBadReadPtr(klass))
    return nullptr;
  if (((uintptr_t)klass & 7) != 0)
    return nullptr;

  return (Il2CppClass *)klass;
}

static bool IsValidIl2CppObject(void *obj)
{
  if (IsBadReadPtr(obj))
    return false;
  if (((uintptr_t)obj & 7) != 0)
    return false; // Object pointer luôn chia hết cho 8 trên 64-bit

  void *klass = *reinterpret_cast<void **>(obj);
  if (IsBadReadPtr(klass))
    return false;
  if (((uintptr_t)klass & 7) != 0)
    return false; // Class pointer luôn chia hết cho 8 trên 64-bit

  const char *className = Il2cpp::GetClassName((Il2CppClass *)klass);
  const char *classNs = Il2cpp::GetClassNamespace((Il2CppClass *)klass);

  if (className && IsBadReadPtr((void *)className))
    return false;
  if (classNs && IsBadReadPtr((void *)classNs))
    return false;

  return true;
}

static void ApplyDragScroll()
{
  if (ImGui::GetScrollMaxY() > 0.0f)
  {
    if (ImGui::IsWindowHovered() && ImGui::IsMouseDragging(0, 3.0f))
    {
      float deltaY = ImGui::GetIO().MouseDelta.y;
      if (deltaY != 0.0f)
      {
        ImGui::SetScrollY(ImGui::GetScrollY() - deltaY);
      }
    }
  }
}

static void RenderRawValueFieldRow(void *rawData, Il2CppClass *klass, FieldInfo *field, bool showTypeColumn, void *tabRootId = nullptr)
{
  if (!rawData || !klass || !field)
    return;
  const char *fieldName = Il2cpp::GetFieldName(field);
  if (!fieldName)
    return;

  if (g_ObjectFieldFilter[0] != '\0')
  {
    std::string fn = fieldName;
    std::string filter = g_ObjectFieldFilter;
    std::transform(fn.begin(), fn.end(), fn.begin(), ::tolower);
    std::transform(filter.begin(), filter.end(), filter.begin(), ::tolower);
    if (fn.find(filter) == std::string::npos)
      return;
  }

  Il2CppType *fieldType = Il2cpp::GetFieldType(field);
  const char *typeName = fieldType ? Il2cpp::GetTypeName(fieldType) : "System.Object";
  std::string cleanType = CleanTypeName(typeName);
  std::string friendlyTypeName = GetFriendlyTypeName(fieldType);
  uint8_t *valPtr = reinterpret_cast<uint8_t *>(rawData) + AdjustValueTypeFieldOffset(Il2cpp::GetFieldOffset(field));

  ImGui::TableNextRow();
  ImGui::TableNextColumn();
  ImGui::TextUnformatted(fieldName);
  ImGui::TableNextColumn();

  Il2CppClass *fieldClass = fieldType ? Il2cpp::GetClassFromType(fieldType) : nullptr;
  bool isEnum = fieldClass ? Il2cpp::GetClassIsEnum(fieldClass) : false;
  ImGui::PushID(field);
  if (isEnum)
  {
    int currentVal = *reinterpret_cast<int *>(valPtr);
    std::vector<std::pair<std::string, int>> enumConstants;
    void *enumIter = nullptr;
    while (auto enumField = Il2cpp::GetClassFields(fieldClass, &enumIter))
    {
      auto flags = Il2cpp::GetFieldFlags(enumField);
      if (flags & FIELD_ATTRIBUTE_LITERAL)
      {
        const char *constName = Il2cpp::GetFieldName(enumField);
        int constVal = 0;
        Il2cpp::GetFieldStaticValue(enumField, &constVal);
        enumConstants.push_back({constName ? constName : "unknown", constVal});
      }
    }
    std::string currentLabel = std::to_string(currentVal);
    for (auto &pair : enumConstants)
    {
      if (pair.second == currentVal)
      {
        currentLabel = pair.first;
        break;
      }
    }
    ImGui::SetNextItemWidth(-FLT_MIN);
    if (ImGui::BeginCombo("##raw_enum", currentLabel.c_str()))
    {
      for (auto &pair : enumConstants)
      {
        bool selected = pair.second == currentVal;
        if (ImGui::Selectable(pair.first.c_str(), selected))
          *reinterpret_cast<int *>(valPtr) = pair.second;
      }
      ImGui::EndCombo();
    }
  }
  else if (IsIntTypeName(cleanType))
  {
    int v = *reinterpret_cast<int *>(valPtr);
    ImGui::SetNextItemWidth(-FLT_MIN);
    if (ImGui::InputInt("##raw_i32", &v, 0, 0))
      *reinterpret_cast<int *>(valPtr) = v;
  }
  else if (IsLongTypeName(cleanType))
  {
    long long v = *reinterpret_cast<int64_t *>(valPtr);
    ImGui::SetNextItemWidth(-FLT_MIN);
    if (ImGui::InputScalar("##raw_i64", ImGuiDataType_S64, &v))
      *reinterpret_cast<int64_t *>(valPtr) = (int64_t)v;
  }
  else if (IsFloatTypeName(cleanType))
  {
    float v = *reinterpret_cast<float *>(valPtr);
    ImGui::SetNextItemWidth(-FLT_MIN);
    if (ImGui::InputFloat("##raw_f32", &v))
      *reinterpret_cast<float *>(valPtr) = v;
  }
  else if (IsDoubleTypeName(cleanType))
  {
    double v = *reinterpret_cast<double *>(valPtr);
    ImGui::SetNextItemWidth(-FLT_MIN);
    if (ImGui::InputDouble("##raw_f64", &v))
      *reinterpret_cast<double *>(valPtr) = v;
  }
  else if (IsBoolTypeName(cleanType))
  {
    bool v = *reinterpret_cast<bool *>(valPtr);
    if (ImGui::Checkbox("##raw_bool", &v))
      *reinterpret_cast<bool *>(valPtr) = v;
  }
  else if (IsStringTypeName(cleanType))
  {
    MonoString *ms = *reinterpret_cast<MonoString **>(valPtr);
    ImGui::TextUnformatted(ms ? ms->toString().c_str() : "null");
  }
  else
  {
    bool isValueType = fieldClass ? Il2cpp::GetClassIsValueType(fieldClass) : false;
    if (isValueType && !isEnum)
    {
      char label[160];
      if (valPtr && !IsBadReadPtr(valPtr))
      {
        snprintf(label, sizeof(label), "%s (Struct)", cleanType.c_str());
        ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.16f, 0.25f, 1.00f));
        if (ImGui::Button(label, ImVec2(-FLT_MIN, 0)))
        {
          if (tabRootId)
            g_TabHistories[tabRootId].push_back({InspectHistoryNode::TYPE_RAW, valPtr, fieldClass, fieldName});
          else
            OpenRawInspectTab(valPtr, fieldClass, fieldName);
        }
        ImGui::PopStyleColor();
      }
      else
      {
        ImGui::TextDisabled("%s (Struct - Bad Pointer)", cleanType.c_str());
      }
    }
    else
    {
      void *subObj = *reinterpret_cast<void **>(valPtr);
      if (subObj && IsValidIl2CppObject(subObj))
      {
        char label[160];
        snprintf(label, sizeof(label), "%s (0x%llX)", cleanType.c_str(), (unsigned long long)subObj);
        if (ImGui::Button(label, ImVec2(-FLT_MIN, 0)))
        {
          if (tabRootId)
            g_TabHistories[tabRootId].push_back({InspectHistoryNode::TYPE_OBJECT, subObj, fieldClass, fieldName});
          else
            OpenInspectedObjectDump(subObj);
        }
      }
      else if (subObj)
      {
        ImGui::TextDisabled("%s (0x%llX - Invalid/GC'd)", cleanType.c_str(), (unsigned long long)subObj);
      }
      else
      {
        ImGui::TextDisabled("%s (null)", cleanType.c_str());
      }
    }
  }
  ImGui::PopID();

  if (showTypeColumn)
  {
    ImGui::TableNextColumn();
    ImGui::TextUnformatted(friendlyTypeName.c_str());
  }
}

static void RenderFieldRow(void *obj, Il2CppClass *klass, FieldEntry &fieldEntry, bool showTypeColumn, void *tabRootId = nullptr)
{
  FieldInfo *field = Il2cpp::GetClassField(klass, fieldEntry.name.c_str());
  if (!field)
    return;
  const char *klassNameRaw = Il2cpp::GetClassName(klass);
  const char *klassNsRaw = Il2cpp::GetClassNamespace(klass);
  std::string ownerClassName = (klassNsRaw && klassNsRaw[0])
                                   ? (std::string(klassNsRaw) + "." + (klassNameRaw ? klassNameRaw : ""))
                                   : (klassNameRaw ? klassNameRaw : "");

  if (g_ObjectFieldFilter[0] != '\0')
  {
    std::string fn = fieldEntry.name;
    std::string filter = g_ObjectFieldFilter;
    std::transform(fn.begin(), fn.end(), fn.begin(), ::tolower);
    std::transform(filter.begin(), filter.end(), filter.begin(), ::tolower);
    if (fn.find(filter) == std::string::npos)
      return;
  }

  ImGui::PushID(field);
  ImGui::TableNextRow();
  ImGui::TableNextColumn();

  ImGui::TextUnformatted(fieldEntry.name.c_str());

  ImGui::TableNextColumn();

  size_t offset = Il2cpp::GetFieldOffset(field);
  bool isStatic = (Il2cpp::GetFieldFlags(field) & FIELD_ATTRIBUTE_STATIC) != 0;

  uint8_t *valPtr = nullptr;
  if (!isStatic)
  {
    valPtr = reinterpret_cast<uint8_t *>(obj) + offset;
  }

  Il2CppType *fieldType = Il2cpp::GetFieldType(field);
  std::string friendlyTypeName = GetFriendlyTypeName(fieldType);
  if (!fieldType)
  {
    ImGui::TextDisabled("Unknown Type");
    if (showTypeColumn)
    {
      ImGui::TableNextColumn();
      ImGui::TextUnformatted(friendlyTypeName.c_str());
    }
    ImGui::PopID();
    return;
  }

  const char *typeName = Il2cpp::GetTypeName(fieldType);
  Il2CppClass *fieldClass = Il2cpp::GetClassFromType(fieldType);
  bool isEnum = fieldClass ? Il2cpp::GetClassIsEnum(fieldClass) : false;

  char widgetId[128];
  snprintf(widgetId, sizeof(widgetId), "##field_%s_%p", fieldEntry.name.c_str(), obj);

  float allW = ImGui::CalcTextSize("All").x + ImGui::GetStyle().FramePadding.x * 2.0f;
  float copyW = ImGui::CalcTextSize("Copy").x + ImGui::GetStyle().FramePadding.x * 2.0f;
  float rightButtonsW = allW + copyW + ImGui::GetStyle().ItemSpacing.x * 2.0f;

  if (isEnum)
  {
    int currentVal = 0;
    {
      std::lock_guard<std::mutex> lock(g_Il2cppMutex);
      if (isStatic)
      {
        Il2cpp::GetFieldStaticValue(field, &currentVal);
      }
      else if (valPtr)
      {
        currentVal = *reinterpret_cast<int *>(valPtr);
      }
    }

    std::vector<std::pair<std::string, int>> enumConstants;
    {
      std::lock_guard<std::mutex> lock(g_Il2cppMutex);
      void *enumIter = nullptr;
      while (auto enumField = Il2cpp::GetClassFields(fieldClass, &enumIter))
      {
        auto flags = Il2cpp::GetFieldFlags(enumField);
        if (flags & FIELD_ATTRIBUTE_LITERAL)
        {
          const char *constName = Il2cpp::GetFieldName(enumField);
          int constVal = 0;
          Il2cpp::GetFieldStaticValue(enumField, &constVal);
          enumConstants.push_back({constName ? constName : "unknown", constVal});
        }
      }
    }

    std::string currentLabel = std::to_string(currentVal);
    for (auto &pair : enumConstants)
    {
      if (pair.second == currentVal)
      {
        currentLabel = pair.first;
        break;
      }
    }

    ImGui::PushStyleColor(ImGuiCol_FrameBg, ImVec4(0.16f, 0.16f, 0.20f, 1.00f));
    ImGui::SetNextItemWidth(isStatic ? -FLT_MIN : (ImGui::GetContentRegionAvail().x - rightButtonsW));
    if (ImGui::BeginCombo(widgetId, currentLabel.c_str()))
    {
      for (auto &pair : enumConstants)
      {
        bool isSelected = (pair.second == currentVal);
        if (ImGui::Selectable(pair.first.c_str(), isSelected))
        {
          if (isStatic)
          {
            Il2cpp::SetFieldStaticValue(field, &pair.second);
          }
          else if (valPtr)
          {
            *reinterpret_cast<int *>(valPtr) = pair.second;
          }
        }
      }
      ImGui::EndCombo();
    }
    if (!isStatic)
    {
      ImGui::SameLine();
      if (ImGui::Button("All##enum", ImVec2(allW, 0)))
      {
        HandleCopyField(tabRootId, ownerClassName, fieldEntry.name, "setFieldInt", std::to_string(currentVal), true);
      }
      ImGui::SameLine();
      if (ImGui::Button("Copy##enum", ImVec2(copyW, 0)))
      {
        HandleCopyField(tabRootId, ownerClassName, fieldEntry.name, "setFieldInt", std::to_string(currentVal));
      }
    }
    ImGui::PopStyleColor();
  }
  else if (strcmp(typeName, "System.Int32") == 0 || strcmp(typeName, "int") == 0)
  {
    int val = 0;
    if (isStatic)
      Il2cpp::GetFieldStaticValue(field, &val);
    else if (valPtr)
      val = *reinterpret_cast<int *>(valPtr);

    ImGui::PushStyleColor(ImGuiCol_FrameBg, ImVec4(0.16f, 0.16f, 0.20f, 1.00f));
    ImGui::SetNextItemWidth(isStatic ? -FLT_MIN : (ImGui::GetContentRegionAvail().x - rightButtonsW));
    if (ImGui::InputInt(widgetId, &val, 0, 0))
    {
      if (isStatic)
        Il2cpp::SetFieldStaticValue(field, &val);
      else if (valPtr)
        *reinterpret_cast<int *>(valPtr) = val;
    }
    if (!isStatic)
    {
      ImGui::SameLine();
      if (ImGui::Button("All##int", ImVec2(allW, 0)))
      {
        HandleCopyField(tabRootId, ownerClassName, fieldEntry.name, "setFieldInt", std::to_string(val), true);
      }
      ImGui::SameLine();
      if (ImGui::Button("Copy##int", ImVec2(copyW, 0)))
      {
        HandleCopyField(tabRootId, ownerClassName, fieldEntry.name, "setFieldInt", std::to_string(val));
      }
    }
    ImGui::PopStyleColor();
  }
  else if (strcmp(typeName, "System.Single") == 0 || strcmp(typeName, "float") == 0)
  {
    float val = 0.0f;
    if (isStatic)
      Il2cpp::GetFieldStaticValue(field, &val);
    else if (valPtr)
      val = *reinterpret_cast<float *>(valPtr);

    ImGui::PushStyleColor(ImGuiCol_FrameBg, ImVec4(0.16f, 0.16f, 0.20f, 1.00f));
    ImGui::SetNextItemWidth(isStatic ? -FLT_MIN : (ImGui::GetContentRegionAvail().x - rightButtonsW));
    if (ImGui::InputFloat(widgetId, &val, 0.0f, 0.0f, "%.3f"))
    {
      if (isStatic)
        Il2cpp::SetFieldStaticValue(field, &val);
      else if (valPtr)
        *reinterpret_cast<float *>(valPtr) = val;
    }
    if (!isStatic)
    {
      ImGui::SameLine();
      if (ImGui::Button("All##float", ImVec2(allW, 0)))
      {
        HandleCopyField(tabRootId, ownerClassName, fieldEntry.name, "setFieldFloat", std::to_string(val), true);
      }
      ImGui::SameLine();
      if (ImGui::Button("Copy##float", ImVec2(copyW, 0)))
      {
        HandleCopyField(tabRootId, ownerClassName, fieldEntry.name, "setFieldFloat", std::to_string(val));
      }
    }
    ImGui::PopStyleColor();
  }
  else if (strcmp(typeName, "System.Boolean") == 0 || strcmp(typeName, "bool") == 0)
  {
    bool val = false;
    if (isStatic)
      Il2cpp::GetFieldStaticValue(field, &val);
    else if (valPtr)
      val = *reinterpret_cast<bool *>(valPtr);

    ImGui::PushStyleColor(ImGuiCol_FrameBg, ImVec4(0.16f, 0.16f, 0.20f, 1.00f));
    ImGui::SetNextItemWidth(isStatic ? -FLT_MIN : (ImGui::GetContentRegionAvail().x - rightButtonsW));
    int intVal = val ? 1 : 0;
    if (ImGui::InputInt(widgetId, &intVal, 0, 0))
    {
      val = (intVal != 0);
      if (isStatic)
        Il2cpp::SetFieldStaticValue(field, &val);
      else if (valPtr)
        *reinterpret_cast<bool *>(valPtr) = val;
    }
    if (!isStatic)
    {
      ImGui::SameLine();
      if (ImGui::Button("All##bool", ImVec2(allW, 0)))
      {
        HandleCopyField(tabRootId, ownerClassName, fieldEntry.name, "setFieldBool", val ? "true" : "false", true);
      }
      ImGui::SameLine();
      if (ImGui::Button("Copy##bool", ImVec2(copyW, 0)))
      {
        HandleCopyField(tabRootId, ownerClassName, fieldEntry.name, "setFieldBool", val ? "true" : "false");
      }
    }
    ImGui::PopStyleColor();
  }
  else if (strcmp(typeName, "System.String") == 0 || strcmp(typeName, "string") == 0)
  {
    MonoString *ms = nullptr;
    if (isStatic)
      Il2cpp::GetFieldStaticValue(field, &ms);
    else if (valPtr)
      ms = *reinterpret_cast<MonoString **>(valPtr);

    std::string strVal = ms ? ms->toString() : "";
    char buf[512];
    strncpy(buf, strVal.c_str(), sizeof(buf));
    buf[sizeof(buf) - 1] = '\0';

    ImGui::PushStyleColor(ImGuiCol_FrameBg, ImVec4(0.16f, 0.16f, 0.20f, 1.00f));
    ImGui::SetNextItemWidth(isStatic ? -FLT_MIN : (ImGui::GetContentRegionAvail().x - rightButtonsW));
    if (ImGui::InputText(widgetId, buf, sizeof(buf)))
    {
      if (isStatic)
      {
        Il2CppString *newStr = Il2cpp::NewString(buf);
        Il2cpp::SetFieldStaticValue(field, &newStr);
        MonoString *verify = nullptr;
        Il2cpp::GetFieldStaticValue(field, &verify);
        LOGI("SetFieldString: field=%s static=1 value=\"%s\" verify=\"%s\"",
             fieldEntry.name.c_str(), buf, verify ? verify->toString().c_str() : "null");
      }
      else
      {
        Il2CppString *newStr = Il2cpp::NewString(buf);
        il2cpp_field_set_value_object(reinterpret_cast<Il2CppObject *>(obj), field, reinterpret_cast<Il2CppObject *>(newStr));
        MonoString *verify = valPtr ? *reinterpret_cast<MonoString **>(valPtr) : nullptr;
        LOGI("SetFieldString: field=%s static=0 offset=0x%llX value=\"%s\" verify=\"%s\"",
             fieldEntry.name.c_str(), (unsigned long long)offset, buf, verify ? verify->toString().c_str() : "null");
      }
    }
    if (!isStatic)
    {
      ImGui::SameLine();
      if (ImGui::Button("All##str", ImVec2(allW, 0)))
      {
        HandleCopyField(tabRootId, ownerClassName, fieldEntry.name, "setFieldString", LuaQuote(buf), true);
      }
      ImGui::SameLine();
      if (ImGui::Button("Copy##str", ImVec2(copyW, 0)))
      {
        HandleCopyField(tabRootId, ownerClassName, fieldEntry.name, "setFieldString", LuaQuote(buf));
      }
    }
    ImGui::PopStyleColor();
  }
  else
  {
    bool isValueType = fieldClass ? Il2cpp::GetClassIsValueType(fieldClass) : false;
    if (isValueType && !isEnum)
    {
      uint8_t *structValPtr = nullptr;
      if (isStatic)
      {
        void *staticData = Il2cpp::GetStaticFieldData(klass);
        if (staticData)
        {
          structValPtr = reinterpret_cast<uint8_t *>(staticData) + offset;
        }
      }
      else
      {
        structValPtr = valPtr;
      }

      char label[160];
      if (structValPtr && !IsBadReadPtr(structValPtr))
      {
        snprintf(label, sizeof(label), "%s (Struct)", CleanTypeName(typeName).c_str());
        ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.16f, 0.25f, 1.00f));
        if (ImGui::Button(label, ImVec2(-FLT_MIN, 0)))
        {
          if (tabRootId)
            g_TabHistories[tabRootId].push_back({InspectHistoryNode::TYPE_RAW, structValPtr, fieldClass, fieldEntry.name});
          else
            OpenRawInspectTab(structValPtr, fieldClass, fieldEntry.name);
        }
        ImGui::PopStyleColor();
      }
      else
      {
        ImGui::TextDisabled("%s (Struct - Bad Pointer)", CleanTypeName(typeName).c_str());
      }
    }
    else
    {
      void *subObj = nullptr;
      if (isStatic)
        Il2cpp::GetFieldStaticValue(field, &subObj);
      else if (valPtr)
        subObj = *reinterpret_cast<void **>(valPtr);

      char label[128];
      if (subObj && IsValidIl2CppObject(subObj))
      {
        snprintf(label, sizeof(label), "%s (0x%llX)", CleanTypeName(typeName).c_str(), (unsigned long long)subObj);
        ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.16f, 0.16f, 0.20f, 1.00f));
        if (ImGui::Button(label, ImVec2(-FLT_MIN, 0)))
        {
          if (tabRootId)
            g_TabHistories[tabRootId].push_back({InspectHistoryNode::TYPE_OBJECT, subObj, fieldClass, fieldEntry.name});
          else
            OpenInspectedObjectDump(subObj);
        }
        ImGui::PopStyleColor();
      }
      else if (subObj)
      {
        snprintf(label, sizeof(label), "%s (0x%llX - Invalid/GC'd)", CleanTypeName(typeName).c_str(), (unsigned long long)subObj);
        ImGui::TextDisabled("%s", label);
      }
      else
      {
        snprintf(label, sizeof(label), "%s (null)", CleanTypeName(typeName).c_str());
        ImGui::TextDisabled("%s", label);
      }
    }
  }

  if (showTypeColumn)
  {
    ImGui::TableNextColumn();
    ImGui::TextUnformatted(friendlyTypeName.c_str());
  }
  ImGui::PopID();
}

static void RenderPropertyRow(void *obj, Il2CppClass *klass, MethodEntry &methodEntry, bool showTypeColumn, void *tabRootId = nullptr)
{
  if (methodEntry.name.rfind("get_", 0) != 0)
    return;

  std::string propName = methodEntry.name.substr(4);
  if (g_ObjectFieldFilter[0] != '\0')
  {
    std::string pn = propName;
    std::string filter = g_ObjectFieldFilter;
    std::transform(pn.begin(), pn.end(), pn.begin(), ::tolower);
    std::transform(filter.begin(), filter.end(), filter.begin(), ::tolower);
    if (pn.find(filter) == std::string::npos)
      return;
  }

  MethodInfo *method = nullptr;
  void *iter = nullptr;
  while (auto m = Il2cpp::GetClassMethods(klass, &iter))
  {
    const char *name = Il2cpp::GetMethodName(m);
    if (name && strcmp(name, methodEntry.name.c_str()) == 0)
    {
      method = m;
      break;
    }
  }
  if (!method)
    return;
  if (Il2cpp::GetMethodParamCount(method) != 0)
    return;
  Il2CppType *returnType = Il2cpp::GetMethodReturnType(method);
  std::string friendlyTypeName = GetFriendlyTypeName(returnType);

  ImGui::TableNextRow();
  ImGui::TableNextColumn();

  ImGui::TextColored(ImVec4(0.8f, 0.5f, 0.8f, 1.0f), "%s (Prop)", propName.c_str());

  ImGui::TableNextColumn();

  Il2CppException *exc = nullptr;
  Il2CppObject *res = il2cpp_runtime_invoke(method, obj, nullptr, &exc);
  if (exc)
  {
    ImGui::TextDisabled("Exception raised");
  }
  else if (!res)
  {
    ImGui::TextDisabled("null");
  }
  else
  {
    Il2CppClass *resClass = SafeGetClass(res);
    const char *resTypeName = resClass ? il2cpp_class_get_name(resClass) : "System.Object";

    if (strcmp(resTypeName, "Int32") == 0 || strcmp(resTypeName, "int") == 0)
    {
      int val = *reinterpret_cast<int *>(il2cpp_object_unbox(res));
      ImGui::Text("%d", val);
    }
    else if (strcmp(resTypeName, "Single") == 0 || strcmp(resTypeName, "float") == 0)
    {
      float val = *reinterpret_cast<float *>(il2cpp_object_unbox(res));
      ImGui::Text("%.3f", val);
    }
    else if (strcmp(resTypeName, "Boolean") == 0 || strcmp(resTypeName, "bool") == 0)
    {
      bool val = *reinterpret_cast<bool *>(il2cpp_object_unbox(res));
      ImGui::Text("%s", val ? "true" : "false");
    }
    else if (strcmp(resTypeName, "String") == 0 || strcmp(resTypeName, "string") == 0)
    {
      MonoString *ms = reinterpret_cast<MonoString *>(res);
      ImGui::Text("%s", ms->toString().c_str());
    }
    else
    {
      char label[128];
      if (res && IsValidIl2CppObject(res))
      {
        snprintf(label, sizeof(label), "%s (0x%llX)", CleanTypeName(resTypeName).c_str(), (unsigned long long)res);
        ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.16f, 0.16f, 0.20f, 1.00f));
        if (ImGui::Button(label, ImVec2(-FLT_MIN, 0)))
        {
          if (tabRootId)
            g_TabHistories[tabRootId].push_back({InspectHistoryNode::TYPE_OBJECT, res, resClass, propName});
          else
            OpenInspectedObjectDump(res);
        }
        ImGui::PopStyleColor();
      }
      else if (res)
      {
        snprintf(label, sizeof(label), "%s (0x%llX - Invalid/GC'd)", CleanTypeName(resTypeName).c_str(), (unsigned long long)res);
        ImGui::TextDisabled("%s", label);
      }
      else
      {
        ImGui::TextDisabled("null");
      }
    }
  }

  if (showTypeColumn)
  {
    ImGui::TableNextColumn();
    ImGui::TextUnformatted(friendlyTypeName.c_str());
  }
}

static bool RenderArrayElementRows(void *obj, Il2CppClass *klass, bool showTypeColumn, void *tabRootId = nullptr)
{
  if (!obj || !klass)
    return false;

  if (IsBadReadPtr(obj) || IsBadReadPtr(klass))
    return false;

  const char *arrayClassName = Il2cpp::GetClassName(klass);
  if (IsBadReadPtr((void *)arrayClassName))
    return false;

  std::string arrayName = arrayClassName ? arrayClassName : "";
  if (arrayName.find("[]") == std::string::npos)
    return false;

  _Il2CppArray *arr = reinterpret_cast<_Il2CppArray *>(obj);
  Il2CppClass *elementClass = il2cpp_class_get_element_class ? il2cpp_class_get_element_class(klass) : nullptr;
  int elemSize = il2cpp_array_element_size ? il2cpp_array_element_size(klass) : 0;
  if (!elementClass || elemSize <= 0)
  {
    ImGui::TableNextRow();
    ImGui::TableNextColumn();
    ImGui::TextUnformatted("Array");
    ImGui::TableNextColumn();
    ImGui::TextDisabled("element metadata unavailable");
    if (showTypeColumn)
    {
      ImGui::TableNextColumn();
      ImGui::TextUnformatted("Array");
    }
    return true;
  }

  uint32_t length = (uint32_t)arr->max_length;
  uint32_t visible = std::min<uint32_t>(length, 256);
  uint8_t *data = reinterpret_cast<uint8_t *>(arr) + sizeof(_Il2CppArray);
  bool elemIsValueType = Il2cpp::GetClassIsValueType(elementClass);
  const char *elemName = Il2cpp::GetClassName(elementClass);
  std::string elemTypeName = elemName ? elemName : "Element";
  std::string elemTypeLower = LowerCopy(elemTypeName);
  bool elemIsBool = elemTypeLower == "boolean" || elemTypeLower == "bool";
  bool elemIsI32 = elemTypeLower == "int32" || elemTypeLower == "int" ||
                   elemTypeLower == "uint32" || elemTypeLower == "short" ||
                   elemTypeLower == "uint16" || elemTypeLower == "byte" ||
                   elemTypeLower == "sbyte";
  bool elemIsI64 = elemTypeLower == "int64" || elemTypeLower == "uint64" || elemTypeLower == "long";
  bool elemIsF32 = elemTypeLower == "single" || elemTypeLower == "float";
  bool elemIsF64 = elemTypeLower == "double";

  for (uint32_t i = 0; i < visible; ++i)
  {
    ImGui::PushID((int)i);
    ImGui::TableNextRow();
    ImGui::TableNextColumn();
    ImGui::Text("[%u]", i);
    ImGui::TableNextColumn();

    if (elemIsValueType)
    {
      void *elemPtr = data + (size_t)i * (size_t)elemSize;
      if (elemIsBool)
      {
        bool v = *reinterpret_cast<bool *>(elemPtr);
        if (ImGui::Checkbox("##arr_bool", &v))
          *reinterpret_cast<bool *>(elemPtr) = v;
      }
      else if (elemIsI32)
      {
        int v = *reinterpret_cast<int *>(elemPtr);
        ImGui::SetNextItemWidth(-FLT_MIN);
        if (ImGui::InputInt("##arr_i32", &v, 0, 0))
          *reinterpret_cast<int *>(elemPtr) = v;
      }
      else if (elemIsI64)
      {
        long long v = *reinterpret_cast<int64_t *>(elemPtr);
        ImGui::SetNextItemWidth(-FLT_MIN);
        if (ImGui::InputScalar("##arr_i64", ImGuiDataType_S64, &v))
          *reinterpret_cast<int64_t *>(elemPtr) = (int64_t)v;
      }
      else if (elemIsF32)
      {
        float v = *reinterpret_cast<float *>(elemPtr);
        ImGui::SetNextItemWidth(-FLT_MIN);
        if (ImGui::InputFloat("##arr_f32", &v))
          *reinterpret_cast<float *>(elemPtr) = v;
      }
      else if (elemIsF64)
      {
        double v = *reinterpret_cast<double *>(elemPtr);
        ImGui::SetNextItemWidth(-FLT_MIN);
        if (ImGui::InputDouble("##arr_f64", &v))
          *reinterpret_cast<double *>(elemPtr) = v;
      }
      else
      {
        char label[192];
        snprintf(label, sizeof(label), "%s @ +0x%llX", elemTypeName.c_str(), (unsigned long long)((uint8_t *)elemPtr - (uint8_t *)arr));
        if (ImGui::Button(label, ImVec2(-FLT_MIN, 0)))
        {
          if (tabRootId)
            g_TabHistories[tabRootId].push_back({InspectHistoryNode::TYPE_RAW, elemPtr, elementClass, "[" + std::to_string(i) + "]"});
          else
            OpenRawInspectTab(elemPtr, elementClass, elemTypeName + "[" + std::to_string(i) + "]");
        }
      }
    }
    else
    {
      void *elemObj = *reinterpret_cast<void **>(data + (size_t)i * (size_t)sizeof(void *));
      if (elemObj && IsValidIl2CppObject(elemObj))
      {
        char label[192];
        snprintf(label, sizeof(label), "%s (0x%llX)", elemTypeName.c_str(), (unsigned long long)elemObj);
        if (ImGui::Button(label, ImVec2(-FLT_MIN, 0)))
        {
          if (tabRootId)
            g_TabHistories[tabRootId].push_back({InspectHistoryNode::TYPE_OBJECT, elemObj, elementClass, "[" + std::to_string(i) + "]"});
          else
            OpenInspectedObjectDump(elemObj);
        }
      }
      else if (elemObj)
      {
        char label[192];
        snprintf(label, sizeof(label), "%s (0x%llX - Invalid/GC'd)", elemTypeName.c_str(), (unsigned long long)elemObj);
        ImGui::TextDisabled("%s", label);
      }
      else
      {
        ImGui::TextDisabled("null");
      }
    }

    if (showTypeColumn)
    {
      ImGui::TableNextColumn();
      ImGui::TextUnformatted(elemIsValueType ? "struct" : "object");
    }
    ImGui::PopID();
  }

  if (visible < length)
  {
    ImGui::TableNextRow();
    ImGui::TableNextColumn();
    ImGui::TextDisabled("...");
    ImGui::TableNextColumn();
    ImGui::TextDisabled("%u more elements hidden", length - visible);
    if (showTypeColumn)
    {
      ImGui::TableNextColumn();
      ImGui::TextUnformatted("Array");
    }
  }
  return true;
}

static void RenderRawStructInspectorContent(void *rawData, Il2CppClass *klass, void *tabRootId = nullptr)
{
  if (!rawData || !klass)
    return;

  if (IsBadReadPtr(rawData) || IsBadReadPtr(klass))
  {
    ImGui::TextColored(ImVec4(1.0f, 0.3f, 0.3f, 1.0f), "Error: Bad Struct/Class Pointer (0x%llX)", (unsigned long long)rawData);
    return;
  }

  float wAvail = ImGui::GetContentRegionAvail().x;
  float spacing = ImGui::GetStyle().ItemSpacing.x;
  float filterLabelW = ImGui::CalcTextSize("Filter").x;
  float inputW = wAvail - filterLabelW - spacing - 6.0f;

  ImGui::SetNextItemWidth(inputW);
  ImGui::InputText("##RawFieldFilter", g_ObjectFieldFilter, sizeof(g_ObjectFieldFilter));
  ImGui::SameLine();
  ImGui::TextUnformatted("Filter");
  ImGui::Separator();

  ImGuiTableFlags tableFlags = ImGuiTableFlags_BordersInnerV |
                               ImGuiTableFlags_BordersOuterV |
                               ImGuiTableFlags_RowBg |
                               ImGuiTableFlags_Resizable |
                               ImGuiTableFlags_ScrollY;
  float tableHeight = std::max(160.0f, ImGui::GetContentRegionAvail().y - 34.0f);
  ImGui::PushStyleVar(ImGuiStyleVar_CellPadding, ImVec2(5.0f, 3.0f));
  if (ImGui::BeginTable("##RawStructFieldsTable", 3, tableFlags, ImVec2(0.0f, tableHeight)))
  {
    ApplyDragScroll();
    float typeColWidth = std::max(58.0f, ImGui::CalcTextSize("Dictionary").x + ImGui::GetStyle().CellPadding.x * 2.0f + 8.0f);
    ImGui::TableSetupColumn("Field/Prop", ImGuiTableColumnFlags_WidthFixed, std::max(150.0f, wAvail * 0.43f));
    ImGui::TableSetupColumn("Value", ImGuiTableColumnFlags_WidthStretch);
    ImGui::TableSetupColumn("Type", ImGuiTableColumnFlags_WidthFixed | ImGuiTableColumnFlags_NoResize, typeColWidth);
    ImGui::TableHeadersRow();

    Il2CppClass *curr = klass;
    while (curr)
    {
      void *iter = nullptr;
      while (auto field = Il2cpp::GetClassFields(curr, &iter))
      {
        if (!field)
          continue;
        if ((Il2cpp::GetFieldFlags(field) & FIELD_ATTRIBUTE_STATIC) != 0)
          continue;
        RenderRawValueFieldRow(rawData, curr, field, true, tabRootId);
      }
      curr = il2cpp_class_get_parent ? il2cpp_class_get_parent(curr) : nullptr;
    }
    ImGui::EndTable();
  }
  ImGui::PopStyleVar();

  if (tabRootId)
  {
    auto &history = g_TabHistories[tabRootId];
    if (history.size() > 1)
    {
      ImGui::Spacing();
      float btnSize = ImGui::GetFrameHeight();
      ImGui::SetCursorPosX(ImGui::GetCursorPosX() + ImGui::GetContentRegionAvail().x - btnSize);
      ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.15f, 0.18f, 0.30f, 1.0f));
      ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.25f, 0.30f, 0.50f, 1.0f));
      if (ImGui::Button("X", ImVec2(btnSize, 0.0f)))
      {
        history.pop_back();
      }
      ImGui::PopStyleColor(2);
    }
  }
}

static void RenderClassInspectorContent(FlatResultEntry &selectedItem);

static FlatResultEntry BuildFlatResultEntryFromClass(Il2CppClass *klass)
{
  FlatResultEntry entry;
  if (!klass)
    return entry;

  const char *ns = Il2cpp::GetClassNamespace(klass);
  const char *name = Il2cpp::GetClassName(klass);
  entry.name = name ? name : "Unknown";
  entry.ns = ns ? ns : "";
  entry.isEnum = Il2cpp::GetClassIsEnum(klass);
  entry.isStruct = Il2cpp::GetClassIsValueType(klass);
  entry.parentName = "";
  if (il2cpp_class_get_parent)
  {
    auto parent = il2cpp_class_get_parent(klass);
    if (parent)
    {
      const char *parentName = Il2cpp::GetClassName(parent);
      if (parentName)
      {
        entry.parentName = parentName;
      }
    }
  }

  uintptr_t il2cppBase = KittyMemory::getLibraryBaseMap("libil2cpp.so").startAddress;

  // Fields
  void *fieldIter = nullptr;
  while (auto field = Il2cpp::GetClassFields(klass, &fieldIter))
  {
    if (!field)
      continue;

    FieldEntry fEntry;
    const char *fieldName = Il2cpp::GetFieldName(field);
    fEntry.name = fieldName ? fieldName : "unknown";

    auto fieldType = Il2cpp::GetFieldType(field);
    const char *fieldTypeName = fieldType ? Il2cpp::GetTypeName(fieldType) : "System.Void";
    fEntry.type = CleanTypeName(fieldTypeName);
    fEntry.offset = Il2cpp::GetFieldOffset(field);
    fEntry.isStatic = (Il2cpp::GetFieldFlags(field) & FIELD_ATTRIBUTE_STATIC) != 0;
    fEntry.signature = FormatFieldSignature(fEntry.type, fEntry.name, fEntry.offset);

    entry.fields.push_back(fEntry);
  }

  // Methods
  void *methodIter = nullptr;
  while (auto method = Il2cpp::GetClassMethods(klass, &methodIter))
  {
    if (!method)
      continue;

    MethodEntry mEntry;
    const char *methodName = Il2cpp::GetMethodName(method);
    mEntry.name = methodName ? methodName : "unknown";

    auto returnType = Il2cpp::GetMethodReturnType(method);
    const char *returnTypeName = returnType ? Il2cpp::GetTypeName(returnType) : "System.Void";
    mEntry.returnType = CleanTypeName(returnTypeName);
    mEntry.paramCount = (int)Il2cpp::GetMethodParamCount(method);
    for (int p = 0; p < mEntry.paramCount; ++p)
    {
      MethodParamEntry paramEntry;
      auto paramType = Il2cpp::GetMethodParam(method, (uint32_t)p);
      const char *paramTypeName = paramType ? Il2cpp::GetTypeName(paramType) : "System.Object";
      const char *paramName = Il2cpp::GetMethodParamName(method, (uint32_t)p);
      paramEntry.type = CleanTypeName(paramTypeName);
      paramEntry.name = (paramName && paramName[0]) ? paramName : ("arg" + std::to_string(p));
      mEntry.params.push_back(paramEntry);
    }

    mEntry.rva = 0;
    mEntry.methodPointer = nullptr;
    if (method->methodPointer)
    {
      mEntry.rva = (uint64_t)method->methodPointer - il2cppBase;
      mEntry.methodPointer = method->methodPointer;
    }

    mEntry.signature = FormatMethodSignature(mEntry.returnType, mEntry.name, method);

    entry.methods.push_back(mEntry);
  }

  return entry;
}

static void RenderObjectInspectorContent(void *obj, Il2CppClass *klass, void *tabRootId = nullptr)
{
  if (!obj || !klass)
    return;

  if (IsBadReadPtr(obj) || IsBadReadPtr(klass))
  {
    ImGui::TextColored(ImVec4(1.0f, 0.3f, 0.3f, 1.0f), "Error: Bad Object/Class Pointer (0x%llX)", (unsigned long long)obj);
    return;
  }

  const char *className = Il2cpp::GetClassName(klass);
  if (IsBadReadPtr((void *)className))
  {
    ImGui::TextColored(ImVec4(1.0f, 0.3f, 0.3f, 1.0f), "Error: Invalid Class Structure");
    return;
  }
  float wAvail = ImGui::GetContentRegionAvail().x;
  float spacing = ImGui::GetStyle().ItemSpacing.x;
  float filterLabelW = ImGui::CalcTextSize("Filter").x;
  float inputW = wAvail - filterLabelW - spacing - 6.0f;
  bool dumpRequested = false;

  ImGui::SetNextItemWidth(inputW);
  ImGui::InputText("##ObjFieldFilter", g_ObjectFieldFilter, sizeof(g_ObjectFieldFilter));
  ImGui::SameLine();
  ImGui::TextUnformatted("Filter");

  ImGui::PushStyleVar(ImGuiStyleVar_FramePadding, ImVec2(5.0f, 3.0f));
  ImGui::Checkbox("Include Properties", &g_IncludeProperties);
  ImGui::SameLine();
  ImGui::Checkbox("Batch Copy Mode", &g_BatchScriptMode);
  if (!g_BatchFieldQueue.empty())
  {
    ImGui::SameLine();
    char batchLabel[64];
    snprintf(batchLabel, sizeof(batchLabel), "Copy Batch (%d)", (int)g_BatchFieldQueue.size());
    if (ImGui::Button(batchLabel))
    {
      std::string batchScript = GenerateBatchScript();
      PublishSetFieldAllScript(batchScript);
    }
    if (ImGui::IsItemHovered())
    {
      std::string tooltipText = "Queued fields:\n";
      for (const auto &f : g_BatchFieldQueue)
      {
        tooltipText += "- " + f.fieldName + " (" + f.className + ")\n";
      }
      ImGui::SetTooltip("%s", tooltipText.c_str());
    }
    ImGui::SameLine();
    if (ImGui::Button("Clear Batch"))
    {
      g_BatchFieldQueue.clear();
    }
  }
  ImGui::PopStyleVar();

  auto dumpObjectToClipboard = [&]()
  {
    std::string dumpText = std::string(className) + " (0x" + std::to_string((unsigned long long)obj) + ")\n{\n";

    Il2CppClass *curr = klass;
    while (curr)
    {
      void *iter = nullptr;
      while (auto field = Il2cpp::GetClassFields(curr, &iter))
      {
        if (!field)
          continue;
        const char *fieldName = Il2cpp::GetFieldName(field);
        if (!fieldName)
          continue;

        bool isStatic = (Il2cpp::GetFieldFlags(field) & FIELD_ATTRIBUTE_STATIC) != 0;
        if (isStatic)
          continue;

        size_t offset = Il2cpp::GetFieldOffset(field);
        uint8_t *valPtr = reinterpret_cast<uint8_t *>(obj) + offset;

        Il2CppType *fieldType = Il2cpp::GetFieldType(field);
        if (!fieldType)
          continue;

        const char *typeName = Il2cpp::GetTypeName(fieldType);

        dumpText += "  " + std::string(fieldName) + " = ";

        Il2CppClass *fieldClass = Il2cpp::GetClassFromType(fieldType);
        bool isEnum = fieldClass ? Il2cpp::GetClassIsEnum(fieldClass) : false;

        if (isEnum)
        {
          int currentVal = *reinterpret_cast<int *>(valPtr);
          std::string label = std::to_string(currentVal);
          void *enumIter = nullptr;
          while (auto enumField = Il2cpp::GetClassFields(fieldClass, &enumIter))
          {
            auto flags = Il2cpp::GetFieldFlags(enumField);
            if (flags & FIELD_ATTRIBUTE_LITERAL)
            {
              int constVal = 0;
              Il2cpp::GetFieldStaticValue(enumField, &constVal);
              if (constVal == currentVal)
              {
                const char *constName = Il2cpp::GetFieldName(enumField);
                if (constName)
                  label = constName;
                break;
              }
            }
          }
          dumpText += label + " (Enum)";
        }
        else if (strcmp(typeName, "System.Int32") == 0 || strcmp(typeName, "int") == 0)
        {
          dumpText += std::to_string(*reinterpret_cast<int *>(valPtr));
        }
        else if (strcmp(typeName, "System.Single") == 0 || strcmp(typeName, "float") == 0)
        {
          dumpText += std::to_string(*reinterpret_cast<float *>(valPtr));
        }
        else if (strcmp(typeName, "System.Boolean") == 0 || strcmp(typeName, "bool") == 0)
        {
          dumpText += *reinterpret_cast<bool *>(valPtr) ? "true" : "false";
        }
        else if (strcmp(typeName, "System.String") == 0 || strcmp(typeName, "string") == 0)
        {
          MonoString *ms = *reinterpret_cast<MonoString **>(valPtr);
          dumpText += "\"" + (ms ? ms->toString() : "") + "\"";
        }
        else
        {
          void *subObj = *reinterpret_cast<void **>(valPtr);
          char subLabel[128];
          snprintf(subLabel, sizeof(subLabel), "0x%llX", (unsigned long long)subObj);
          dumpText += std::string(CleanTypeName(typeName)) + " (" + subLabel + ")";
        }
        dumpText += "\n";
      }
      curr = il2cpp_class_get_parent ? il2cpp_class_get_parent(curr) : nullptr;
    }
    dumpText += "}";
    CopyToClipboard(dumpText.c_str());
  };

  ImGui::Separator();

  bool showTypeColumn = true;
  int tableColumns = 3;
  ImGuiTableFlags tableFlags = ImGuiTableFlags_BordersInnerV |
                               ImGuiTableFlags_BordersOuterV |
                               ImGuiTableFlags_RowBg |
                               ImGuiTableFlags_Resizable;
  ImGui::PushStyleVar(ImGuiStyleVar_CellPadding, ImVec2(5.0f, 3.0f));
  ImGui::PushStyleColor(ImGuiCol_TableHeaderBg, ImVec4(0.11f, 0.13f, 0.21f, 0.95f));
  ImGui::PushStyleColor(ImGuiCol_TableRowBg, ImVec4(0.05f, 0.06f, 0.09f, 0.50f));
  ImGui::PushStyleColor(ImGuiCol_TableRowBgAlt, ImVec4(0.07f, 0.09f, 0.14f, 0.70f));
  if (ImGui::BeginTable("##ObjectFieldsTable", tableColumns, tableFlags, ImVec2(0.0f, 0.0f)))
  {
    ApplyDragScroll();
    float typeColWidth = ImGui::CalcTextSize("Dictionary").x + ImGui::GetStyle().CellPadding.x * 2.0f + 8.0f;
    typeColWidth = std::max(58.0f, typeColWidth);
    ImGui::TableSetupColumn("Field/Prop", ImGuiTableColumnFlags_WidthFixed, std::max(150.0f, wAvail * 0.43f));
    ImGui::TableSetupColumn("Value", ImGuiTableColumnFlags_WidthStretch);
    ImGui::TableSetupColumn("Type", ImGuiTableColumnFlags_WidthFixed | ImGuiTableColumnFlags_NoResize, typeColWidth);
    ImGui::TableHeadersRow();

    bool renderedArray = RenderArrayElementRows(obj, klass, showTypeColumn, tabRootId);
    Il2CppClass *curr = klass;
    while (curr && !renderedArray)
    {
      void *iter = nullptr;
      while (auto field = Il2cpp::GetClassFields(curr, &iter))
      {
        if (!field)
          continue;

        const char *fieldName = Il2cpp::GetFieldName(field);
        if (!fieldName)
          continue;

        FieldEntry fEntry;
        fEntry.name = fieldName;

        auto fieldType = Il2cpp::GetFieldType(field);
        const char *fieldTypeName = fieldType ? Il2cpp::GetTypeName(fieldType) : "System.Void";
        fEntry.type = CleanTypeName(fieldTypeName);
        fEntry.offset = Il2cpp::GetFieldOffset(field);
        fEntry.isStatic = (Il2cpp::GetFieldFlags(field) & FIELD_ATTRIBUTE_STATIC) != 0;

        RenderFieldRow(obj, curr, fEntry, showTypeColumn, tabRootId);
      }

      if (g_IncludeProperties)
      {
        void *mIter = nullptr;
        while (auto method = Il2cpp::GetClassMethods(curr, &mIter))
        {
          if (!method)
            continue;

          const char *methodName = Il2cpp::GetMethodName(method);
          if (!methodName || strncmp(methodName, "get_", 4) != 0)
            continue;

          MethodEntry mEntry;
          mEntry.name = methodName;

          RenderPropertyRow(obj, curr, mEntry, showTypeColumn, tabRootId);
        }
      }

      curr = il2cpp_class_get_parent ? il2cpp_class_get_parent(curr) : nullptr;
    }

    ImGui::EndTable();
  }
  ImGui::PopStyleColor(3);
  ImGui::PopStyleVar();

  if (tabRootId)
  {
    auto &history = g_TabHistories[tabRootId];
    if (history.size() > 1)
    {
      ImGui::Spacing();
      ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.25f, 0.43f, 1.00f));
      ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.28f, 0.35f, 0.60f, 1.00f));
      if (ImGui::Button("<- Back to Previous Object", ImVec2(-FLT_MIN, 0)))
      {
        history.pop_back();
      }
      ImGui::PopStyleColor(2);
    }
  }

  ImGui::Spacing();
  ImGui::Separator();

  // Cơ chế Cache Map tĩnh để tránh gọi IL2CPP API liên tục mỗi frame
  static std::unordered_map<Il2CppClass *, FlatResultEntry> s_ClassEntryCache;
  auto it = s_ClassEntryCache.find(klass);
  if (it == s_ClassEntryCache.end())
  {
    bool foundInCache = false;
    const char *ns = Il2cpp::GetClassNamespace(klass);
    const char *name = Il2cpp::GetClassName(klass);
    std::string nsStr = ns ? ns : "";
    std::string nameStr = name ? name : "";

    FlatResultEntry entry;
    // Thử tìm trong g_MetadataCache trước (nếu đã load metadata)
    for (const auto &img : g_MetadataCache)
    {
      for (const auto &cls : img.classes)
      {
        if (cls.name == nameStr && cls.ns == nsStr)
        {
          entry.name = cls.name;
          entry.ns = cls.ns;
          entry.isEnum = cls.isEnum;
          entry.isStruct = cls.isStruct;
          entry.parentName = cls.parentName;
          entry.methods = cls.methods;
          entry.fields = cls.fields;
          foundInCache = true;
          break;
        }
      }
      if (foundInCache)
        break;
    }

    if (!foundInCache)
    {
      // Nếu không tìm thấy, mới thực hiện build lâm thời đúng 1 lần duy nhất
      entry = BuildFlatResultEntryFromClass(klass);
    }

    s_ClassEntryCache[klass] = entry;
    it = s_ClassEntryCache.find(klass);
  }

  FlatResultEntry &currentClassEntry = it->second;
  ImGui::TextColored(ImVec4(0.29f, 0.37f, 0.65f, 1.0f), "Class Inspector: %s", currentClassEntry.name.c_str());
  ImGui::Spacing();

  RenderClassInspectorContent(currentClassEntry);
}

static void RenderClassInspectorContent(FlatResultEntry &selectedItem)
{
  std::string classFullName = BuildClassFullName(selectedItem);
  std::string classDeclaration = BuildClassDeclaration(selectedItem);

  ImGui::PushStyleColor(ImGuiCol_Header, ImVec4(0.17f, 0.22f, 0.36f, 1.0f));
  ImGui::PushStyleColor(ImGuiCol_HeaderHovered, ImVec4(0.24f, 0.31f, 0.51f, 1.0f));
  ImGui::PushStyleColor(ImGuiCol_HeaderActive, ImVec4(0.30f, 0.39f, 0.64f, 1.0f));

  bool nodeOpen = ImGui::TreeNodeEx(selectedItem.name.c_str(), ImGuiTreeNodeFlags_Framed | ImGuiTreeNodeFlags_DefaultOpen);
  RenderCopyPopupForLastItem(("class_" + classFullName).c_str(), "Copy Class", {{"Full Name", classFullName}, {"Declaration", classDeclaration}, {"Name", selectedItem.name}, {"Namespace", selectedItem.ns}});

  if (nodeOpen)
  {

    std::vector<FieldEntry> instanceFields;
    std::vector<FieldEntry> staticFields;
    for (const auto &field : selectedItem.fields)
    {
      if (field.isStatic)
        staticFields.push_back(field);
      else
        instanceFields.push_back(field);
    }

    ImGui::Unindent();

    if (!instanceFields.empty())
    {
      if (ImGui::TreeNodeEx("Fields", ImGuiTreeNodeFlags_Framed))
      {
        for (size_t i = 0; i < instanceFields.size(); ++i)
        {
          const auto &field = instanceFields[i];
          if (g_ObjectFieldFilter[0] != '\0')
          {
            std::string hay = field.type + " " + field.name + " " + field.signature;
            std::string needle = g_ObjectFieldFilter;
            std::transform(hay.begin(), hay.end(), hay.begin(), ::tolower);
            std::transform(needle.begin(), needle.end(), needle.begin(), ::tolower);
            if (hay.find(needle) == std::string::npos)
              continue;
          }
          if (g_FilterSimpleMode)
          {
            ImGui::Text("  %s %s", field.type.c_str(), field.name.c_str());
          }
          else
          {
            ImGui::Text("  public %s", field.signature.c_str());
          }
          std::string fieldId = "field_inst_" + classFullName + "_" + std::to_string(i);
          std::string fieldCopy = "public " + field.signature;
          RenderCopyPopupForLastItem(fieldId.c_str(), "Copy Field", {{"Name", field.name}, {"Signature", fieldCopy}, {"Offset", "0x" + [&]()
                                                                                                                                    {
                                                                                                                                      char buf[32];
                                                                                                                                      snprintf(buf, sizeof(buf), "%zX", field.offset);
                                                                                                                                      return std::string(buf);
                                                                                                                                    }()},
                                                                     {"Class", classFullName}});
        }
        ImGui::TreePop();
      }
    }

    if (!staticFields.empty())
    {
      if (ImGui::TreeNodeEx("Static Fields", ImGuiTreeNodeFlags_Framed))
      {
        for (size_t i = 0; i < staticFields.size(); ++i)
        {
          const auto &field = staticFields[i];
          if (g_ObjectFieldFilter[0] != '\0')
          {
            std::string hay = field.type + " " + field.name + " " + field.signature;
            std::string needle = g_ObjectFieldFilter;
            std::transform(hay.begin(), hay.end(), hay.begin(), ::tolower);
            std::transform(needle.begin(), needle.end(), needle.begin(), ::tolower);
            if (hay.find(needle) == std::string::npos)
              continue;
          }
          if (g_FilterSimpleMode)
          {
            ImGui::Text("  %s %s", field.type.c_str(), field.name.c_str());
          }
          else
          {
            ImGui::Text("  public %s", field.signature.c_str());
          }
          std::string fieldId = "field_static_" + classFullName + "_" + std::to_string(i);
          std::string fieldCopy = "public " + field.signature;
          RenderCopyPopupForLastItem(fieldId.c_str(), "Copy Field", {{"Name", field.name}, {"Signature", fieldCopy}, {"Offset", "0x" + [&]()
                                                                                                                                    {
                                                                                                                                      char buf[32];
                                                                                                                                      snprintf(buf, sizeof(buf), "%zX", field.offset);
                                                                                                                                      return std::string(buf);
                                                                                                                                    }()},
                                                                     {"Class", classFullName}});
        }
        ImGui::TreePop();
      }
    }

    ImGui::Indent();

    if (!selectedItem.methods.empty())
    {
      std::vector<PatchEntry> patches = LuaConsole_GetActivePatchList();
      for (size_t methodIndex = 0; methodIndex < selectedItem.methods.size(); ++methodIndex)
      {
        auto &method = selectedItem.methods[methodIndex];
        if (g_ObjectFieldFilter[0] != '\0')
        {
          std::string hay = method.returnType + " " + method.name + " " + method.signature;
          std::string needle = g_ObjectFieldFilter;
          std::transform(hay.begin(), hay.end(), hay.begin(), ::tolower);
          std::transform(needle.begin(), needle.end(), needle.begin(), ::tolower);
          if (hay.find(needle) == std::string::npos)
            continue;
        }
        const PatchEntry *patch = FindRedirectPatchForMethod(patches, classFullName, method.name);
        bool hasRedirect = patch != nullptr;
        bool enabledRedirect = patch && patch->enabled;
        std::string methodLabel;
        if (g_FilterSimpleMode)
        {
          methodLabel = method.signature + "; // RVA: 0x" + [&]()
          {
            char buf[32];
            snprintf(buf, sizeof(buf), "%llX", (unsigned long long)method.rva);
            return std::string(buf);
          }();
        }
        else
        {
          methodLabel = "public " + method.signature + "; // RVA: 0x" + [&]()
          {
            char buf[32];
            snprintf(buf, sizeof(buf), "%llX", (unsigned long long)method.rva);
            return std::string(buf);
          }();
        }

        bool isTraced = false;
        if (method.methodPointer)
        {
          void *ptr = UNTAG_PTR(method.methodPointer);
          std::lock_guard<std::mutex> lock(g_TraceMutex);
          auto it = g_TracedMethodsMap.find(ptr);
          if (it != g_TracedMethodsMap.end() && it->second.enabled)
          {
            isTraced = true;
          }
        }

        ImGui::PushID((int)methodIndex);
        if (isTraced)
        {
          ImGui::PushStyleColor(ImGuiCol_Text, ImVec4(0.25f, 1.0f, 0.35f, 1.0f));
        }
        else if (hasRedirect)
        {
          ImGui::PushStyleColor(ImGuiCol_Text, enabledRedirect ? ImVec4(0.25f, 1.0f, 0.35f, 1.0f) : ImVec4(0.55f, 0.55f, 0.58f, 1.0f));
        }
        if (g_LastSelectedClassFullName == classFullName && g_LastSelectedMethodName == method.name)
          ImGui::SetNextItemOpen(true, ImGuiCond_Once);
        bool open = ImGui::TreeNodeEx(methodLabel.c_str(), ImGuiTreeNodeFlags_SpanAvailWidth);
        if (ImGui::IsItemClicked() && g_LastSelectedMethodName != method.name)
        {
          g_LastSelectedClassFullName = classFullName;
          g_LastSelectedMethodName = method.name;
          SaveConfig();
        }
        if (isTraced || hasRedirect)
        {
          ImGui::PopStyleColor();
        }

        std::string methodId = "method_" + classFullName + "_" + std::to_string(methodIndex);
        std::string scriptTemplate = BuildRedirectScript(classFullName, method, classFullName, method);
        RenderCopyPopupForLastItem(methodId.c_str(), "Copy Method", {{"Name", method.name}, {"Signature", method.signature}, {"RVA", "0x" + [&]()
                                                                                                                                         {
                                                                                                                                           char buf[32];
                                                                                                                                           snprintf(buf, sizeof(buf), "%llX", (unsigned long long)method.rva);
                                                                                                                                           return std::string(buf);
                                                                                                                                         }()},
                                                                     {"Class", classFullName},
                                                                     {"Redirect Script", scriptTemplate}});

        if (open)
        {
          if (ImGui::BeginTabBar("##MethodTabs"))
          {
            if (ImGui::BeginTabItem("Caller"))
            {
              static int callIntValues[16] = {};
              static long long callLongValues[16] = {};
              static float callFloatValues[16] = {};
              static double callDoubleValues[16] = {};
              static bool callBoolValues[16] = {};
              static char callStringValues[16][128] = {};
              static char callExprValues[16][160] = {};

              std::string thisSlotKey = methodId + "_this";
              uintptr_t callerSelectedObject = g_CallerObjectSelections[thisSlotKey];
              ImGui::Text("%s this", selectedItem.name.c_str());

              std::string comboPreview = callerSelectedObject ? FormatObjectAddress(callerSelectedObject) : "";
              ImGui::SetNextItemWidth(-FLT_MIN);
              if (ImGui::BeginCombo(("##this_combo_" + methodId).c_str(), comboPreview.c_str()))
              {
                std::string classFullName = BuildClassFullName(selectedItem);
                OpenCallerObjectPicker(classFullName, selectedItem.name, thisSlotKey);
                ImGui::EndCombo();
              }
              ImGui::Spacing();

              std::vector<std::string> argExprs;
              argExprs.reserve(method.params.size());
              int maxParams = std::min((int)method.params.size(), 16);
              for (int p = 0; p < maxParams; ++p)
              {
                const auto &param = method.params[(size_t)p];
                std::string label = param.type + " " + param.name;
                ImGui::PushID(p);

                if (IsBoolTypeName(param.type))
                {
                  int choice = callBoolValues[p] ? 1 : 0;
                  const char *items[] = {"false", "true"};
                  ImGui::Text("%s", label.c_str());
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  if (ImGui::Combo(("##combo_bool_" + std::to_string(p)).c_str(), &choice, items, 2))
                    callBoolValues[p] = choice != 0;
                  argExprs.push_back(callBoolValues[p] ? "true" : "false");
                }
                else if (IsIntTypeName(param.type))
                {
                  ImGui::Text("%s", label.c_str());
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  ImGui::InputInt(("##input_int_" + std::to_string(p)).c_str(), &callIntValues[p]);
                  argExprs.push_back(std::to_string(callIntValues[p]));
                }
                else if (IsLongTypeName(param.type))
                {
                  ImGui::Text("%s", label.c_str());
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  ImGui::InputScalar(("##input_long_" + std::to_string(p)).c_str(), ImGuiDataType_S64, &callLongValues[p]);
                  argExprs.push_back(std::to_string(callLongValues[p]));
                }
                else if (IsFloatTypeName(param.type))
                {
                  ImGui::Text("%s", label.c_str());
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  ImGui::InputFloat(("##input_float_" + std::to_string(p)).c_str(), &callFloatValues[p]);
                  argExprs.push_back(std::to_string(callFloatValues[p]));
                }
                else if (IsDoubleTypeName(param.type))
                {
                  ImGui::Text("%s", label.c_str());
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  ImGui::InputDouble(("##input_double_" + std::to_string(p)).c_str(), &callDoubleValues[p]);
                  argExprs.push_back(std::to_string(callDoubleValues[p]));
                }
                else if (IsStringTypeName(param.type))
                {
                  ImGui::Text("%s", label.c_str());
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  ImGui::InputText(("##input_str_" + std::to_string(p)).c_str(), callStringValues[p], sizeof(callStringValues[p]));
                  argExprs.push_back(LuaQuote(callStringValues[p]));
                }
                else
                {
                  std::string paramSlotKey = methodId + "_arg_" + std::to_string(p);
                  uintptr_t selectedParamObj = g_CallerObjectSelections[paramSlotKey];
                  std::string argComboPreview = selectedParamObj ? FormatObjectAddress(selectedParamObj) : "";

                  ImGui::Text("%s", label.c_str());
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  if (ImGui::BeginCombo(("##combo_obj_" + paramSlotKey).c_str(), argComboPreview.c_str()))
                  {
                    OpenCallerObjectPicker(param.type, label, paramSlotKey);
                    ImGui::EndCombo();
                  }
                  argExprs.push_back(selectedParamObj ? ObjectExprFromAddress(selectedParamObj) : "nil");
                }
                ImGui::PopID();
                ImGui::Spacing();
              }
              if ((int)method.params.size() > maxParams)
              {
                ImGui::TextDisabled("Only first %d params are editable.", maxParams);
              }

              int &callCount = g_CallerCallCounts[methodId];
              if (callCount <= 0)
                callCount = 1;

              std::string callScript = BuildCallScript(classFullName, method, argExprs, callerSelectedObject, callCount);
              float callerW = ImGui::GetContentRegionAvail().x;
              if (ImGui::Button("Call in Main Thread", ImVec2(callerW, 0)))
              {
                LuaConsole_SetScriptText(callScript.c_str());
                g_PendingCallerResultKey = methodId;
                LuaConsole_RequestExecute(callScript.c_str());
              }

              ImGui::Spacing();
              ImGui::AlignTextToFramePadding();

              char callCountBuf[16];
              snprintf(callCountBuf, sizeof(callCountBuf), "%d", callCount);
              ImGui::SetNextItemWidth(45.0f);
              if (ImGui::InputText("##CallCountVal", callCountBuf, sizeof(callCountBuf), ImGuiInputTextFlags_CharsDecimal))
              {
                int val = atoi(callCountBuf);
                if (val >= 1 && val <= 100)
                  callCount = val;
              }

              ImGui::SameLine();
              if (ImGui::Button("-##dec", ImVec2(ImGui::GetFrameHeight(), 0.0f)))
              {
                if (callCount > 1)
                  callCount--;
              }

              ImGui::SameLine();
              if (ImGui::Button("+##inc", ImVec2(ImGui::GetFrameHeight(), 0.0f)))
              {
                if (callCount < 100)
                  callCount++;
              }

              ImGui::SameLine();
              ImGui::TextUnformatted("Call Count");

              ImGui::SameLine();
              float copyBtnW = ImGui::CalcTextSize("Copy Script").x + ImGui::GetStyle().FramePadding.x * 2.0f + 8.0f;
              float availX = ImGui::GetContentRegionAvail().x;
              ImGui::SetCursorPosX(ImGui::GetCursorPosX() + availX - copyBtnW);
              if (ImGui::Button("Copy Script", ImVec2(copyBtnW, 0)))
              {
                CopyToClipboard(callScript.c_str());
              }

              ImGui::SeparatorText("Call Results");
              auto resultIt = g_CallerResultValues.find(methodId);
              auto rawIt = g_CallerResultRaw.find(methodId);
              bool hasValues = resultIt != g_CallerResultValues.end() && !resultIt->second.empty();
              bool hasRaw = rawIt != g_CallerResultRaw.end() && !rawIt->second.empty();
              if (hasValues || hasRaw)
              {
                if (hasValues)
                {
                  for (size_t resultIndex = 0; resultIndex < resultIt->second.size(); ++resultIndex)
                  {
                    RenderCopyableResultValue(resultIt->second[resultIndex], method.returnType,
                                              ("call_result_" + std::to_string(resultIndex)).c_str());
                  }
                }
                if (hasRaw)
                {
                  ImGui::PushTextWrapPos(ImGui::GetCursorPosX() + ImGui::GetContentRegionAvail().x);
                  ImGui::TextUnformatted(rawIt->second.c_str());
                  ImGui::PopTextWrapPos();
                }
                if (ImGui::SmallButton("Clear Results"))
                {
                  g_CallerResultValues.erase(methodId);
                  g_CallerResultRaw.erase(methodId);
                }
              }
              else
              {
                ImGui::TextDisabled("No output yet.");
              }
              ImGui::EndTabItem();
            }

            if (ImGui::BeginTabItem("Patcher"))
            {
              if (hasRedirect)
              {
                bool isReturnPatch = patch->type.rfind("Return", 0) == 0;
                float patchW = ImGui::GetContentRegionAvail().x;
                float patchSpacing = ImGui::GetStyle().ItemSpacing.x;
                float patchBtnW = std::max(90.0f, (patchW - patchSpacing) / 2.0f);

                if (ImGui::Button("Restore", ImVec2(patchBtnW, 0)))
                {
                  LuaConsole_PatchReturn(patch->nativeKey, patch->type);
                }
                ImGui::SameLine();
                if (patch->enabled)
                {
                  if (ImGui::Button("Disable", ImVec2(patchBtnW, 0)))
                  {
                    LuaConsole_PatchToggleEnable(patch->nativeKey, patch->type);
                  }
                }
                else
                {
                  if (ImGui::Button("Enable", ImVec2(patchBtnW, 0)))
                  {
                    LuaConsole_PatchToggleEnable(patch->nativeKey, patch->type);
                  }
                }

                if (ImGui::Button("Copy Script", ImVec2(-FLT_MIN, 0)))
                {
                  std::string script;
                  if (isReturnPatch)
                  {
                    std::string luaFunc = GetReturnLuaFunctionFromPatchType(patch->type);
                    std::string valueExpr = patch->type == "ReturnString" ? LuaQuote(patch->targetMethod) : patch->targetMethod;
                    script = BuildReturnScript(classFullName, method, luaFunc, valueExpr);
                  }
                  else
                  {
                    script = "Hook.redirectDirect(" +
                             LuaQuote(classFullName) + ", " +
                             LuaQuote(method.name) + ", " +
                             (classFullName == patch->targetClass
                                  ? LuaQuote(patch->targetMethod)
                                  : LuaQuote(patch->targetClass) + ", " + LuaQuote(patch->targetMethod)) +
                             ")";
                  }
                  CopyToClipboard(script.c_str());
                }
                if (isReturnPatch)
                {
                  ImGui::TextDisabled("%s %s = %s",
                                      patch->type.c_str(),
                                      method.name.c_str(),
                                      patch->targetMethod.c_str());
                }
                else
                {
                  ImGui::TextDisabled("%s -> %s.%s",
                                      classFullName.c_str(),
                                      patch->targetClass.c_str(),
                                      patch->targetMethod.c_str());
                }
              }
              else
              {
                if (ImGui::Button("Redirect", ImVec2(-FLT_MIN, 0)))
                {
                  g_RedirectChooserId = methodId;
                  g_RedirectSourceClass = classFullName;
                  g_RedirectSourceMethod = method;
                  g_ShowRedirectChooserWindow = true;
                  g_FitRedirectChooserWindowNextOpen = true;
                }
              }

              ImGui::Separator();
              ImGui::TextUnformatted("Return Value");
              std::string returnLuaFunc = GetReturnLuaFunction(method.returnType);
              if (returnLuaFunc.empty())
              {
                ImGui::TextDisabled("Unsupported return type: %s", method.returnType.c_str());
              }
              else if (method.paramCount != 0)
              {
                ImGui::TextDisabled("Return hook supports only 0-argument methods/getters.");
              }
              else
              {
                static bool returnBoolValue = true;
                static int returnIntValue = 0;
                static long long returnLongValue = 0;
                static float returnFloatValue = 0.0f;
                static double returnDoubleValue = 0.0;
                static char returnStringValue[256] = "";

                std::string valueExpr;
                if (returnLuaFunc == "returnBool")
                {
                  int boolChoice = returnBoolValue ? 1 : 0;
                  const char *items[] = {"false", "true"};
                  ImGui::TextUnformatted("Value");
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  if (ImGui::Combo("##retBool", &boolChoice, items, 2))
                    returnBoolValue = boolChoice != 0;
                  valueExpr = returnBoolValue ? "true" : "false";
                }
                else if (returnLuaFunc == "returnString")
                {
                  ImGui::TextUnformatted("Value");
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  ImGui::InputText("##retString", returnStringValue, sizeof(returnStringValue));
                  valueExpr = LuaQuote(returnStringValue);
                }
                else if (returnLuaFunc == "returnLong")
                {
                  ImGui::TextUnformatted("Value");
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  ImGui::InputScalar("##retLong", ImGuiDataType_S64, &returnLongValue);
                  valueExpr = std::to_string(returnLongValue);
                }
                else if (returnLuaFunc == "returnFloat")
                {
                  ImGui::TextUnformatted("Value");
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  ImGui::InputFloat("##retFloat", &returnFloatValue);
                  valueExpr = std::to_string(returnFloatValue);
                }
                else if (returnLuaFunc == "returnDouble")
                {
                  ImGui::TextUnformatted("Value");
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  ImGui::InputDouble("##retDouble", &returnDoubleValue);
                  valueExpr = std::to_string(returnDoubleValue);
                }
                else
                {
                  ImGui::TextUnformatted("Value");
                  ImGui::SetNextItemWidth(-FLT_MIN);
                  ImGui::InputInt("##retInt", &returnIntValue);
                  valueExpr = std::to_string(returnIntValue);
                }

                std::string returnScript = BuildReturnScript(classFullName, method, returnLuaFunc, valueExpr);
                float retW = ImGui::GetContentRegionAvail().x;
                float retSpacing = ImGui::GetStyle().ItemSpacing.x;
                float retBtnW = std::max(90.0f, (retW - retSpacing) / 2.0f);

                if (ImGui::Button("Run Return", ImVec2(retBtnW, 0)))
                {
                  LuaConsole_SetScriptText(returnScript.c_str());
                  LuaConsole_RequestExecute(returnScript.c_str());
                  g_ActiveTabIndex = 1;
                }
                ImGui::SameLine();
                if (ImGui::Button("Copy Script##returnScript", ImVec2(retBtnW, 0)))
                {
                  CopyToClipboard(returnScript.c_str());
                }
              }
              ImGui::EndTabItem();
            }
            ImGui::EndTabBar();
          }
          ImGui::TreePop();
        }
        ImGui::Separator();
        ImGui::PopID();
      }
    }
    ImGui::TreePop();
  }
  ImGui::PopStyleColor(3);
}

static void UpdateFilteredMetadataUI()
{
  g_FilteredList.clear();

  std::string patternStr = g_PatternSearch;
  std::string filterStr = g_FilterSearch;

  int activeFilterType = g_FilterType;

  // Smart search shortcut: if query starts with ':', force Method mode
  if (!patternStr.empty() && patternStr[0] == ':')
  {
    activeFilterType = 1;
    patternStr = patternStr.substr(1);
  }
  if (!filterStr.empty() && filterStr[0] == ':')
  {
    activeFilterType = 1;
    filterStr = filterStr.substr(1);
  }

  bool hasPattern = !patternStr.empty();
  bool hasFilter = !filterStr.empty();

  if (!g_FilterCaseSensitive)
  {
    std::transform(patternStr.begin(), patternStr.end(), patternStr.begin(), ::tolower);
    std::transform(filterStr.begin(), filterStr.end(), filterStr.begin(), ::tolower);
  }

  auto matches = [&](const std::string &name, const std::string &query) -> bool
  {
    if (query.empty())
      return true;
    std::string n = name;
    if (!g_FilterCaseSensitive)
    {
      std::transform(n.begin(), n.end(), n.begin(), ::tolower);
    }

    if (g_FilterStartsWith)
    {
      return n.rfind(query, 0) == 0;
    }
    if (g_FilterEndsWith)
    {
      if (n.length() < query.length())
        return false;
      return n.compare(n.length() - query.length(), query.length(), query) == 0;
    }
    return n.find(query) != std::string::npos;
  };

  for (size_t imgIdx = 0; imgIdx < g_MetadataCache.size(); ++imgIdx)
  {
    if (!g_SearchAllImages && (int)imgIdx != g_SelectedImageIdx)
      continue;

    auto &img = g_MetadataCache[imgIdx];

    for (auto &cls : img.classes)
    {
      bool classFilterMatches = true;
      if (hasFilter && activeFilterType == 0)
      {
        classFilterMatches = matches(cls.name, filterStr) || matches(cls.ns, filterStr);
      }

      if (!classFilterMatches)
        continue;

      if (activeFilterType == 0) // Class mode
      {
        if (!hasPattern || matches(cls.name, patternStr) || matches(cls.ns, patternStr))
        {
          FlatResultEntry entry;
          entry.name = cls.name;
          entry.ns = cls.ns;
          entry.isEnum = cls.isEnum;
          entry.isStruct = cls.isStruct;
          entry.parentName = cls.parentName;
          entry.methods = cls.methods;
          entry.fields = cls.fields;
          g_FilteredList.push_back(entry);
        }
      }
      else if (activeFilterType == 1) // Method mode
      {
        std::vector<MethodEntry> matchingMethods;
        for (auto &m : cls.methods)
        {
          std::string searchTarget = m.name;
          if (g_FilterMethodOption == 0)
            searchTarget = m.returnType;
          else if (g_FilterMethodOption == 2)
          {
            searchTarget = "";
            for (const auto &p : m.params)
              searchTarget += p.type + " " + p.name + ", ";
          }

          bool patternMatch = !hasPattern || matches(searchTarget, patternStr);
          bool filterMatch = !hasFilter || matches(searchTarget, filterStr);

          if (patternMatch && filterMatch)
          {
            matchingMethods.push_back(m);
          }
        }

        if (!matchingMethods.empty())
        {
          FlatResultEntry entry;
          entry.name = cls.name;
          entry.ns = cls.ns;
          entry.isEnum = cls.isEnum;
          entry.isStruct = cls.isStruct;
          entry.parentName = cls.parentName;
          entry.methods = matchingMethods;
          entry.fields = cls.fields;
          g_FilteredList.push_back(entry);
        }
      }
      else // Field mode
      {
        std::vector<FieldEntry> matchingFields;
        for (auto &f : cls.fields)
        {
          bool patternMatch = !hasPattern || matches(f.name, patternStr);
          bool filterMatch = !hasFilter || matches(f.name, filterStr);

          if (patternMatch && filterMatch)
          {
            matchingFields.push_back(f);
          }
        }

        if (!matchingFields.empty())
        {
          FlatResultEntry entry;
          entry.name = cls.name;
          entry.ns = cls.ns;
          entry.isEnum = cls.isEnum;
          entry.isStruct = cls.isStruct;
          entry.parentName = cls.parentName;
          entry.fields = matchingFields;
          entry.methods = cls.methods;
          g_FilteredList.push_back(entry);
        }
      }
    }
  }
}

static void RefreshMetadataCache()
{
  g_MetadataCache.clear();
  g_TotalImages = 0;
  g_TotalClasses = 0;
  g_TotalMethods = 0;
  g_TotalFields = 0;

  if (!Il2cpp::EnsureAttached())
  {
    return;
  }

  auto &images = Il2cpp::GetImages();
  if (images.empty())
  {
    return;
  }

  uintptr_t il2cppBase = KittyMemory::getLibraryBaseMap("libil2cpp.so").startAddress;

  for (auto image : images)
  {
    if (!image)
      continue;

    ImageEntry imgEntry;
    const char *imgName = Il2cpp::GetImageName(image);
    imgEntry.name = imgName ? imgName : "Unknown";
    g_TotalImages++;

    auto classes = Il2cpp::GetClasses(image);
    for (auto klass : classes)
    {
      if (!klass)
        continue;

      ClassEntry clsEntry;
      const char *ns = Il2cpp::GetClassNamespace(klass);
      const char *name = Il2cpp::GetClassName(klass);
      clsEntry.name = name ? name : "Unknown";
      clsEntry.ns = ns ? ns : "";
      clsEntry.isEnum = Il2cpp::GetClassIsEnum(klass);
      clsEntry.isStruct = Il2cpp::GetClassIsValueType(klass);
      clsEntry.parentName = "";
      if (il2cpp_class_get_parent)
      {
        auto parent = il2cpp_class_get_parent(klass);
        if (parent)
        {
          const char *parentName = Il2cpp::GetClassName(parent);
          if (parentName)
          {
            clsEntry.parentName = parentName;
          }
        }
      }
      g_TotalClasses++;

      // Fields
      void *fieldIter = nullptr;
      while (auto field = Il2cpp::GetClassFields(klass, &fieldIter))
      {
        if (!field)
          continue;

        FieldEntry fEntry;
        const char *fieldName = Il2cpp::GetFieldName(field);
        fEntry.name = fieldName ? fieldName : "unknown";

        auto fieldType = Il2cpp::GetFieldType(field);
        const char *fieldTypeName = fieldType ? Il2cpp::GetTypeName(fieldType) : "System.Void";
        fEntry.type = CleanTypeName(fieldTypeName);
        fEntry.offset = Il2cpp::GetFieldOffset(field);
        fEntry.isStatic = (Il2cpp::GetFieldFlags(field) & FIELD_ATTRIBUTE_STATIC) != 0;
        fEntry.signature = FormatFieldSignature(fEntry.type, fEntry.name, fEntry.offset);

        clsEntry.fields.push_back(fEntry);
        g_TotalFields++;
      }

      // Methods
      void *methodIter = nullptr;
      while (auto method = Il2cpp::GetClassMethods(klass, &methodIter))
      {
        if (!method)
          continue;

        MethodEntry mEntry;
        const char *methodName = Il2cpp::GetMethodName(method);
        mEntry.name = methodName ? methodName : "unknown";

        auto returnType = Il2cpp::GetMethodReturnType(method);
        const char *returnTypeName = returnType ? Il2cpp::GetTypeName(returnType) : "System.Void";
        mEntry.returnType = CleanTypeName(returnTypeName);
        mEntry.paramCount = (int)Il2cpp::GetMethodParamCount(method);
        for (int p = 0; p < mEntry.paramCount; ++p)
        {
          MethodParamEntry paramEntry;
          auto paramType = Il2cpp::GetMethodParam(method, (uint32_t)p);
          const char *paramTypeName = paramType ? Il2cpp::GetTypeName(paramType) : "System.Object";
          const char *paramName = Il2cpp::GetMethodParamName(method, (uint32_t)p);
          paramEntry.type = CleanTypeName(paramTypeName);
          paramEntry.name = (paramName && paramName[0]) ? paramName : ("arg" + std::to_string(p));
          mEntry.params.push_back(paramEntry);
        }

        mEntry.rva = 0;
        mEntry.methodPointer = nullptr;
        if (method->methodPointer)
        {
          mEntry.rva = (uint64_t)method->methodPointer - il2cppBase;
          mEntry.methodPointer = method->methodPointer;
        }

        mEntry.signature = FormatMethodSignature(mEntry.returnType, mEntry.name, method);

        clsEntry.methods.push_back(mEntry);
        g_TotalMethods++;
      }

      imgEntry.classes.push_back(clsEntry);
    }

    g_MetadataCache.push_back(imgEntry);
  }

  g_MetadataLoaded = true;
  g_FilterNeedsUpdate = true;
}

static bool DumpMetadataAPI(const std::string &outPathDir, int &classCount)
{
  if (!Il2cpp::EnsureAttached())
  {
    return false;
  }

  auto &images = Il2cpp::GetImages();
  if (images.empty())
  {
    return false;
  }

  g_DumpStatusMsg = "Writing dump.cs...";

  std::string outPath = outPathDir + "dump.cs";
  std::ofstream outStream(outPath, std::ios::binary);
  if (!outStream.is_open())
  {
    return false;
  }

  classCount = 0;

  // Get base address of libil2cpp.so using KittyMemory
  uintptr_t il2cppBase = KittyMemory::getLibraryBaseMap("libil2cpp.so").startAddress;

  for (auto image : images)
  {
    if (!image)
      continue;
    const char *imageName = Il2cpp::GetImageName(image);
    outStream << "// Image: " << (imageName ? imageName : "Unknown") << "\n";

    auto classes = Il2cpp::GetClasses(image);
    for (auto klass : classes)
    {
      if (!klass)
        continue;

      classCount++;
      const char *ns = Il2cpp::GetClassNamespace(klass);
      const char *name = Il2cpp::GetClassName(klass);

      outStream << "\n// Namespace: " << (ns ? ns : "") << "\n";

      bool is_valuetype = Il2cpp::GetClassIsValueType(klass);
      bool is_enum = Il2cpp::GetClassIsEnum(klass);

      if (is_enum)
      {
        outStream << "public enum " << (name ? name : "Unknown") << "\n{\n";
      }
      else if (is_valuetype)
      {
        outStream << "public struct " << (name ? name : "Unknown") << "\n{\n";
      }
      else
      {
        outStream << "public class " << (name ? name : "Unknown");
        if (il2cpp_class_get_parent)
        {
          auto parent = il2cpp_class_get_parent(klass);
          if (parent)
          {
            const char *parentName = Il2cpp::GetClassName(parent);
            if (parentName)
            {
              outStream << " : " << parentName;
            }
          }
        }
        outStream << "\n{\n";
      }

      // Fields
      void *fieldIter = nullptr;
      while (auto field = Il2cpp::GetClassFields(klass, &fieldIter))
      {
        if (!field)
          continue;
        const char *fieldName = Il2cpp::GetFieldName(field);
        auto fieldType = Il2cpp::GetFieldType(field);
        const char *fieldTypeName = fieldType ? Il2cpp::GetTypeName(fieldType) : "System.Void";
        size_t offset = Il2cpp::GetFieldOffset(field);

        outStream << "    public " << CleanTypeName(fieldTypeName) << " " << (fieldName ? fieldName : "unknown")
                  << "; // Offset: 0x" << std::hex << offset << "\n";
      }

      outStream << "\n";

      // Methods
      void *methodIter = nullptr;
      while (auto method = Il2cpp::GetClassMethods(klass, &methodIter))
      {
        if (!method)
          continue;
        const char *methodName = Il2cpp::GetMethodName(method);
        auto returnType = Il2cpp::GetMethodReturnType(method);
        const char *returnTypeName = returnType ? Il2cpp::GetTypeName(returnType) : "System.Void";

        uint64_t rva = 0;
        if (method->methodPointer)
        {
          rva = (uint64_t)method->methodPointer - il2cppBase;
        }

        outStream << "    // RVA: 0x" << std::hex << rva << " Offset: 0x" << std::hex << rva << "\n";
        outStream << "    public " << CleanTypeName(returnTypeName) << " " << (methodName ? methodName : "unknown") << "(";

        int paramCount = Il2cpp::GetMethodParamCount(method);
        for (int p = 0; p < paramCount; p++)
        {
          auto paramType = Il2cpp::GetMethodParam(method, p);
          const char *paramTypeName = paramType ? Il2cpp::GetTypeName(paramType) : "System.Void";
          const char *paramName = Il2cpp::GetMethodParamName(method, p);
          outStream << CleanTypeName(paramTypeName) << " " << (paramName ? paramName : "");
          if (p < paramCount - 1)
          {
            outStream << ", ";
          }
        }
        outStream << ");\n";
      }

      outStream << "}\n";
    }
  }

  outStream.close();
  return true;
}

static uintptr_t FindMetadataInMemory(size_t &outSize)
{
  auto maps = KittyMemory::getAllMaps();
  for (const auto &map : maps)
  {
    if (map.readable && map.length >= sizeof(int32_t))
    {
      int32_t magic = 0;
      if (KittyMemory::memRead(&magic, (const void *)map.startAddress, sizeof(magic)))
      {
        if (magic == (int32_t)0xFAB11BAF)
        {
          int32_t version = 0;
          if (KittyMemory::memRead(&version, (const void *)(map.startAddress + 4), sizeof(version)))
          {
            if (version >= 24 && version <= 31)
            {
              outSize = map.length;
              return map.startAddress;
            }
          }
        }
      }
    }
  }

  for (const auto &map : maps)
  {
    if (map.readable && map.length >= 0x1000 && map.pathname.find("base.apk") != std::string::npos)
    {
      for (uintptr_t addr = map.startAddress; addr < map.endAddress - 0x100; addr += 4096)
      {
        int32_t magic = 0;
        if (KittyMemory::memRead(&magic, (const void *)addr, sizeof(magic)))
        {
          if (magic == (int32_t)0xFAB11BAF)
          {
            int32_t version = 0;
            if (KittyMemory::memRead(&version, (const void *)(addr + 4), sizeof(version)))
            {
              if (version >= 24 && version <= 31)
              {
                outSize = map.endAddress - addr;
                return addr;
              }
            }
          }
        }
      }
    }
  }
  return 0;
}

class MetadataParser
{
public:
  const char *metadataBytes;
  size_t metadataSize;
  int32_t version;

  int32_t stringOffset;
  int32_t stringCount;
  int32_t typeDefinitionsOffset;
  int32_t typeDefinitionsCount;
  int32_t methodsOffset;
  int32_t methodsCount;
  int32_t fieldsOffset;
  int32_t fieldsCount;

  MetadataParser(const char *bytes, size_t size) : metadataBytes(bytes), metadataSize(size)
  {
    version = *(int32_t *)(bytes + 4);

    stringOffset = *(int32_t *)(bytes + 24);
    stringCount = *(int32_t *)(bytes + 28);
    methodsOffset = *(int32_t *)(bytes + 48);
    methodsCount = *(int32_t *)(bytes + 52);
    fieldsOffset = *(int32_t *)(bytes + 96);
    fieldsCount = *(int32_t *)(bytes + 100);
    typeDefinitionsOffset = *(int32_t *)(bytes + 160);
    typeDefinitionsCount = *(int32_t *)(bytes + 164);

    if (version == 27)
    {
      typeDefinitionsOffset = *(int32_t *)(bytes + 176);
      typeDefinitionsCount = *(int32_t *)(bytes + 180);
    }
    else if (version == 29)
    {
      typeDefinitionsOffset = *(int32_t *)(bytes + 184);
      typeDefinitionsCount = *(int32_t *)(bytes + 188);
    }
    else if (version == 31)
    {
      typeDefinitionsOffset = *(int32_t *)(bytes + 208);
      typeDefinitionsCount = *(int32_t *)(bytes + 212);
    }
  }

  const char *GetString(int32_t index)
  {
    if (index < 0 || index >= stringCount)
      return "Unknown";
    return (const char *)(metadataBytes + stringOffset + index);
  }

  struct TypeDef
  {
    const char *name;
    const char *ns;
    int32_t parentIndex;
    int32_t methodStart;
    int32_t fieldStart;
    uint16_t methodCount;
    uint16_t fieldCount;
  };

  TypeDef GetTypeDef(int32_t index)
  {
    TypeDef td = {"Unknown", "Unknown", -1, 0, 0, 0, 0};
    size_t structSize = (version >= 29) ? 88 : 92;
    const char *ptr = metadataBytes + typeDefinitionsOffset + (index * structSize);

    td.name = GetString(*(int32_t *)ptr);
    td.ns = GetString(*(int32_t *)(ptr + 4));
    td.parentIndex = *(int32_t *)(ptr + 20);
    td.fieldStart = *(int32_t *)(ptr + 44);
    td.methodStart = *(int32_t *)(ptr + 48);
    td.methodCount = *(uint16_t *)(ptr + 76);
    td.fieldCount = *(uint16_t *)(ptr + 80);
    return td;
  }
};

static bool DumpMetadataFallback(const char *metadataBytes, size_t metadataSize, const std::string &outPathDir, int &classCount)
{
  MetadataParser parser(metadataBytes, metadataSize);

  g_DumpStatusMsg = "Writing dump.cs...";

  std::string outPath = outPathDir + "dump.cs";
  std::ofstream outStream(outPath, std::ios::binary);
  if (!outStream.is_open())
  {
    return false;
  }

  outStream << "// Dumped via fallback parser (stripped symbols)\n";
  outStream << "// Metadata Version: " << parser.version << "\n\n";

  classCount = parser.typeDefinitionsCount / ((parser.version >= 29) ? 88 : 92);

  for (int i = 0; i < classCount; i++)
  {
    auto td = parser.GetTypeDef(i);
    outStream << "\n// Namespace: " << td.ns << "\n";
    outStream << "public class " << td.name << "\n{\n";

    // Fields
    size_t fieldStructSize = 12; // typically 12 bytes
    const char *fieldsPtr = metadataBytes + parser.fieldsOffset;
    for (int f = 0; f < td.fieldCount; f++)
    {
      int32_t fieldIndex = td.fieldStart + f;
      const char *fieldName = parser.GetString(*(int32_t *)(fieldsPtr + fieldIndex * fieldStructSize));
      outStream << "    public object " << fieldName << ";\n";
    }

    outStream << "\n";

    // Methods
    size_t methodStructSize = (parser.version >= 29) ? 36 : 40;
    const char *methodsPtr = metadataBytes + parser.methodsOffset;
    for (int m = 0; m < td.methodCount; m++)
    {
      int32_t methodIndex = td.methodStart + m;
      const char *methodName = parser.GetString(*(int32_t *)(methodsPtr + methodIndex * methodStructSize));
      outStream << "    // RVA: 0x0 Offset: 0x0\n";
      outStream << "    public void " << methodName << "();\n";
    }

    outStream << "}\n";
  }

  outStream.close();
  return true;
}

static void StartDump(const std::string &folderPath)
{
  g_DumpStatus = 1; // Parsing metadata...
  g_DumpStatusMsg = "Parsing metadata...";

  std::thread([folderPath]()
              {
    std::string outPathDir = folderPath;
    if (outPathDir.back() != '/')
    {
      outPathDir += "/";
    }
    
    int classCount = 0;
    bool success = false;
    
    // 1. Try using the API-based dumper first
    if (il2cpp_domain_get != nullptr)
    {
      success = DumpMetadataAPI(outPathDir, classCount);
    }
    
    // 2. Fallback to parsing from memory
    if (!success)
    {
      size_t metadataSize = 0;
      uintptr_t metadataAddr = FindMetadataInMemory(metadataSize);
      if (metadataAddr != 0 && metadataSize > 0)
      {
        success = DumpMetadataFallback((const char*)metadataAddr, metadataSize, outPathDir, classCount);
      }
    }
    
    if (success)
    {
      static char statusMsg[128] = "";
      snprintf(statusMsg, sizeof(statusMsg), "Done! %d classes dumped", classCount);
      g_DumpStatusMsg = statusMsg;
      g_DumpStatus = 2; // Done
    }
    else
    {
      g_DumpStatus = 3; // Failed
    } })
      .detach();
}

static void OpenFindObjectsWindowForClass(const FlatResultEntry &selectedItem)
{
  std::string classFullName = BuildClassFullName(selectedItem);
  if (g_ScanClassName != classFullName)
  {
    g_ScannedObjects.clear();
    g_FindObjectsFilter[0] = '\0';
  }
  g_ScanClassName = classFullName;
  g_ShowInstancesWindow = true;
  g_FitInstancesWindowNextOpen = true;
}

static void RenderInspectorTabSet(const char *tabBarId, FlatResultEntry &selectedItem)
{
  if (!ImGui::BeginTabBar(tabBarId, ImGuiTabBarFlags_Reorderable | ImGuiTabBarFlags_FittingPolicyScroll))
    return;

  if (ImGui::BeginTabItem(selectedItem.name.c_str()))
  {
    RenderClassInspectorContent(selectedItem);
    ImGui::EndTabItem();
  }

  int removeIndex = -1;
  int removeRawIndex = -1;
  for (int i = 0; i < (int)g_InspectedObjectTabs.size(); ++i)
  {
    void *obj = g_InspectedObjectTabs[i];
    if (!obj)
    {
      removeIndex = i;
      continue;
    }

    Il2CppClass *klass = SafeGetClass(obj);
    const char *className = klass ? Il2cpp::GetClassName(klass) : "Unknown";
    char tabName[128];
    snprintf(tabName, sizeof(tabName), "Inspecting %s 0x%llX", className, (unsigned long long)obj);

    bool open = true;
    ImGuiTabItemFlags objectTabFlags = (g_SelectInspectedObjectTab && g_InspectedObject == obj)
                                           ? ImGuiTabItemFlags_SetSelected
                                           : 0;
    ImGui::PushID(obj);
    if (ImGui::BeginTabItem(tabName, &open, objectTabFlags))
    {
      g_InspectedObject = obj;
      g_SelectInspectedObjectTab = false;

      auto &history = g_TabHistories[obj];
      if (history.empty())
      {
        history.push_back({InspectHistoryNode::TYPE_OBJECT, obj, klass, className});
      }

      RenderInspectBreadcrumbs(history);

      auto &currentNode = history.back();
      if (currentNode.type == InspectHistoryNode::TYPE_OBJECT)
      {
        RenderObjectInspectorContent(currentNode.ptr, currentNode.klass, obj);
      }
      else
      {
        RenderRawStructInspectorContent(currentNode.ptr, currentNode.klass, obj);
      }

      ImGui::EndTabItem();
    }
    ImGui::PopID();

    if (!open)
    {
      removeIndex = i;
      if (g_InspectedObject == obj)
        g_InspectedObject = nullptr;
    }
  }

  for (int i = 0; i < (int)g_RawInspectTabs.size(); ++i)
  {
    RawInspectTab &tab = g_RawInspectTabs[i];
    if (!tab.data || !tab.klass)
    {
      removeRawIndex = i;
      continue;
    }

    char tabName[160];
    snprintf(tabName, sizeof(tabName), "Inspecting %s 0x%llX", tab.label.c_str(), (unsigned long long)tab.data);
    bool open = true;
    ImGuiTabItemFlags rawFlags = (g_SelectRawInspectTab == i) ? ImGuiTabItemFlags_SetSelected : 0;
    ImGui::PushID((int)(100000 + i));
    if (ImGui::BeginTabItem(tabName, &open, rawFlags))
    {
      g_SelectRawInspectTab = -1;

      auto &history = g_TabHistories[tab.data];
      if (history.empty())
      {
        history.push_back({InspectHistoryNode::TYPE_RAW, tab.data, tab.klass, tab.label});
      }

      RenderInspectBreadcrumbs(history);

      auto &currentNode = history.back();
      if (currentNode.type == InspectHistoryNode::TYPE_OBJECT)
      {
        RenderObjectInspectorContent(currentNode.ptr, currentNode.klass, tab.data);
      }
      else
      {
        RenderRawStructInspectorContent(currentNode.ptr, currentNode.klass, tab.data);
      }

      ImGui::EndTabItem();
    }
    ImGui::PopID();

    if (!open)
      removeRawIndex = i;
  }

  if (ImGui::TabItemButton("+", ImGuiTabItemFlags_Trailing | ImGuiTabItemFlags_NoTooltip))
  {
    OpenFindObjectsWindowForClass(selectedItem);
  }
  if (ImGui::IsItemHovered())
    ImGui::SetTooltip("Add object tab");

  ImGui::EndTabBar();

  if (removeIndex >= 0 && removeIndex < (int)g_InspectedObjectTabs.size())
  {
    void *removedObj = g_InspectedObjectTabs[removeIndex];
    g_InspectedObjectTabs.erase(g_InspectedObjectTabs.begin() + removeIndex);
    g_TabHistories.erase(removedObj);
    if (!g_InspectedObject && !g_InspectedObjectTabs.empty())
    {
      g_InspectedObject = g_InspectedObjectTabs.back();
      g_SelectInspectedObjectTab = true;
    }
  }

  if (removeRawIndex >= 0 && removeRawIndex < (int)g_RawInspectTabs.size())
  {
    void *removedData = g_RawInspectTabs[removeRawIndex].data;
    g_RawInspectTabs.erase(g_RawInspectTabs.begin() + removeRawIndex);
    g_TabHistories.erase(removedData);
    if (g_SelectRawInspectTab >= (int)g_RawInspectTabs.size())
      g_SelectRawInspectTab = -1;
  }
}

static bool CheckSeedKey(const char *input)
{
  if (strcmp(input, "V7") == 0)
  {
    g_KeyUnlocked = true;
    g_SilentMode = false;
    g_KeyInvalid = false;
    return true;
  }
  else if (strcmp(input, "cilent") == 0 || strcmp(input, "client") == 0)
  {
    g_KeyUnlocked = true;
    g_SilentMode = true;
    g_KeyInvalid = false;
    return true;
  }
  g_KeyInvalid = true;
  return false;
}

static void RenderKeyGate()
{
  ImGui::TextUnformatted("Enter Access Key:");
  ImGui::Spacing();

  // Automatically focus keyboard input on the field
  static bool s_needFocus = true;
  if (s_needFocus)
  {
    ImGui::SetKeyboardFocusHere();
    s_needFocus = false;
  }

  float availW = ImGui::GetContentRegionAvail().x;
  float spacing = ImGui::GetStyle().ItemSpacing.x;
  float loginBtnW = 80.0f;
  float inputW = availW - loginBtnW - spacing;

  ImGui::SetNextItemWidth(inputW);
  bool hitEnter = ImGui::InputText("##KeyInput", g_KeyInput, sizeof(g_KeyInput), ImGuiInputTextFlags_Password | ImGuiInputTextFlags_EnterReturnsTrue);

  ImGui::SameLine();
  ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.3f, 0.6f, 0.3f, 1.0f));
  ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.4f, 0.7f, 0.4f, 1.0f));
  if (ImGui::Button("Login", ImVec2(loginBtnW, 0)) || hitEnter)
  {
    if (CheckSeedKey(g_KeyInput))
    {
      strncpy(g_SavedKey, g_KeyInput, sizeof(g_SavedKey));
      SaveConfig();
      s_needFocus = true; // reset focus trigger for future lock sessions
    }
  }
  ImGui::PopStyleColor(2);

  if (g_KeyInvalid)
  {
    ImGui::Spacing();
    ImGui::PushStyleColor(ImGuiCol_Text, ImVec4(1.0f, 0.35f, 0.35f, 1.0f));
    ImGui::TextUnformatted("Invalid key");
    ImGui::PopStyleColor();
  }
}

static void TraceCallback(RegisterContext *ctx, const HookEntryInfo *info)
{
  void *address = info->function_address;

  std::lock_guard<std::mutex> lock(g_TraceMutex);
  auto it = g_TracedMethodsMap.find(address);
  if (it == g_TracedMethodsMap.end())
    return;

  TraceState &state = it->second;
  if (!state.enabled)
    return;

  auto now = std::chrono::steady_clock::now();
  // Reset hit count if it hasn't been called for more than 10 seconds (new call burst)
  if (std::chrono::duration_cast<std::chrono::milliseconds>(now - state.lastTime).count() > 10000)
  {
    state.hitCount = 0;
  }
  // Spam protection: reset count if > 500ms since last call
  if (std::chrono::duration_cast<std::chrono::milliseconds>(now - state.lastTime).count() > 500)
  {
    state.countInPeriod = 0;
  }
  state.lastTime = now;
  state.countInPeriod++;

  if (g_TraceLimitEnabled)
  {
    if (state.countInPeriod > g_TraceMaxSpam)
    {
      if (g_TraceInstantRemove)
      {
        state.enabled = false;
      }
      return;
    }
  }

  state.hitCount++;

  TraceLogEntry entry;
  entry.methodPointer = state.methodPointer;
  entry.className = state.className;
  entry.methodName = state.methodName;
  entry.fullName = state.fullName;

  g_TraceLogs.push_back(entry);
  if (g_TraceLogs.size() > 10000)
  {
    g_TraceLogs.erase(g_TraceLogs.begin());
  }
}

static void TraceClassMethods(FlatResultEntry &selectedItem)
{
  std::string classFullName = BuildClassFullName(selectedItem);
  for (auto &method : selectedItem.methods)
  {
    if (!method.methodPointer)
      continue;
    void *ptr = UNTAG_PTR(method.methodPointer);

    std::lock_guard<std::mutex> lock(g_TraceMutex);
    auto it = g_TracedMethodsMap.find(ptr);
    if (it == g_TracedMethodsMap.end())
    {
      TraceState state;
      state.methodPointer = ptr;
      state.rva = method.rva;
      state.className = selectedItem.name;
      state.methodName = method.name;
      state.fullName = classFullName + "::" + method.name;
      state.hitCount = 0;
      state.enabled = true;
      state.lastTime = std::chrono::steady_clock::now();
      state.countInPeriod = 0;

      g_TracedMethodsMap[ptr] = state;
      g_TracedMethodsOrder.push_back(ptr);

      DobbyInstrument(ptr, TraceCallback);
    }
    else
    {
      it->second.enabled = true;
    }
  }
}

static void RestoreClassMethods(FlatResultEntry &selectedItem)
{
  for (auto &method : selectedItem.methods)
  {
    if (!method.methodPointer)
      continue;
    void *ptr = UNTAG_PTR(method.methodPointer);

    std::lock_guard<std::mutex> lock(g_TraceMutex);
    auto it = g_TracedMethodsMap.find(ptr);
    if (it != g_TracedMethodsMap.end())
    {
      DobbyDestroy(ptr);
      g_TracedMethodsMap.erase(it);

      auto orderIt = std::find(g_TracedMethodsOrder.begin(), g_TracedMethodsOrder.end(), ptr);
      if (orderIt != g_TracedMethodsOrder.end())
      {
        g_TracedMethodsOrder.erase(orderIt);
      }
    }
  }
}

static void DrawTraceOverlay()
{
  std::lock_guard<std::mutex> lock(g_TraceMutex);
  if (g_TracedMethodsOrder.empty())
    return;

  // Draw borderless transparent window at top-left
  ImGui::SetNextWindowPos(ImVec2(10.0f, 10.0f), ImGuiCond_Once);
  ImGuiWindowFlags flags = ImGuiWindowFlags_NoDecoration |
                           ImGuiWindowFlags_NoSavedSettings |
                           ImGuiWindowFlags_NoScrollbar |
                           ImGuiWindowFlags_NoBackground |
                           ImGuiWindowFlags_NoInputs |
                           ImGuiWindowFlags_AlwaysAutoResize;

  if (ImGui::Begin("##TraceOverlay", nullptr, flags))
  {
    auto now = std::chrono::steady_clock::now();
    for (void *ptr : g_TracedMethodsOrder)
    {
      auto it = g_TracedMethodsMap.find(ptr);
      if (it != g_TracedMethodsMap.end() && it->second.hitCount > 0)
      {
        // Skip drawing if last called >= 10 seconds ago
        if (std::chrono::duration_cast<std::chrono::seconds>(now - it->second.lastTime).count() >= 10)
        {
          continue;
        }

        std::string text;
        if (g_TraceDisplayClassName)
        {
          text = it->second.className + "::" + it->second.methodName;
        }
        else
        {
          text = it->second.methodName;
        }
        text += " " + std::to_string(it->second.hitCount) + "x";
        ImGui::TextColored(ImVec4(0.3f, 0.8f, 0.3f, 1.0f), "%s", text.c_str());
      }
    }
  }
  ImGui::End();
}

static void RenderTraceTabContent()
{
  float availW = ImGui::GetContentRegionAvail().x;
  float itemSpacing = ImGui::GetStyle().ItemSpacing.x;

  // First row: Max input and limit checkbox
  ImGui::AlignTextToFramePadding();
  ImGui::SetNextItemWidth(120.0f * g_FontScale);
  ImGui::InputInt("##TraceMaxSpamInput", &g_TraceMaxSpam, 0, 0);
  ImGui::SameLine();
  if (ImGui::Button("-", ImVec2(ImGui::GetFrameHeight(), 0.0f)))
  {
    g_TraceMaxSpam = std::max(0, g_TraceMaxSpam - 10);
  }
  ImGui::SameLine();
  if (ImGui::Button("+", ImVec2(ImGui::GetFrameHeight(), 0.0f)))
  {
    g_TraceMaxSpam += 10;
  }
  ImGui::SameLine();
  ImGui::TextUnformatted("Max");
  ImGui::SameLine();
  ImGui::Checkbox("Limit", &g_TraceLimitEnabled);

  // Second row: Instant Remove
  ImGui::Checkbox("Instant Remove After Limit", &g_TraceInstantRemove);

  // Third row: Display Class Name and Display Address
  ImGui::Checkbox("Display Class Name", &g_TraceDisplayClassName);
  ImGui::SameLine();
  ImGui::Checkbox("Display Address", &g_TraceDisplayAddress);

  ImGui::Spacing();

  // Fourth row: Mode Toggle Button (Pink/Purple)
  ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.25f, 0.43f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.28f, 0.35f, 0.60f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_ButtonActive, ImVec4(0.35f, 0.44f, 0.75f, 1.00f));
  std::string modeText = (g_TraceMode == 0) ? "Traced Methods" : "Trace Logs";
  if (ImGui::Button(modeText.c_str(), ImVec2(-FLT_MIN, 0.0f)))
  {
    g_TraceMode = (g_TraceMode == 0) ? 1 : 0;
  }
  ImGui::PopStyleColor(3);

  ImGui::Spacing();

  // Separator text and content rendering
  char sepText[128];
  if (g_TraceMode == 0)
  {
    snprintf(sepText, sizeof(sepText), "Traced Methods (x%d)", (int)g_TracedMethodsOrder.size());
    ImGui::SeparatorText(sepText);

    // Filter input box
    static char traceFilterBuf[128] = "";
    float filterLabelW = ImGui::CalcTextSize("Filter").x;
    ImGui::SetNextItemWidth(std::max(100.0f, availW - filterLabelW - itemSpacing * 2.0f));
    ImGui::InputText("##TraceMethodFilter", traceFilterBuf, sizeof(traceFilterBuf));
    ImGui::SameLine();
    ImGui::TextUnformatted("Filter");

    ImGui::Spacing();

    // Scrollable list
    ImVec2 listSize(0.0f, ImGui::GetContentRegionAvail().y - 45.0f);
    ImGui::BeginChild("##TracedMethodsList", listSize, true);

    std::vector<void *> methodsToDraw = g_TracedMethodsOrder;
    if (g_TraceSortByHit)
    {
      std::sort(methodsToDraw.begin(), methodsToDraw.end(), [](void *a, void *b)
                {
        std::lock_guard<std::mutex> lock(g_TraceMutex);
        return g_TracedMethodsMap[a].hitCount > g_TracedMethodsMap[b].hitCount; });
    }

    std::vector<void *> toRestore;
    for (size_t i = 0; i < methodsToDraw.size(); ++i)
    {
      void *ptr = methodsToDraw[i];
      g_TraceMutex.lock();
      auto it = g_TracedMethodsMap.find(ptr);
      if (it == g_TracedMethodsMap.end())
      {
        g_TraceMutex.unlock();
        continue;
      }
      auto state = it->second;
      g_TraceMutex.unlock();

      // Filter matching
      if (traceFilterBuf[0] != '\0')
      {
        std::string hay = state.fullName;
        std::string needle = traceFilterBuf;
        std::transform(hay.begin(), hay.end(), hay.begin(), ::tolower);
        std::transform(needle.begin(), needle.end(), needle.begin(), ::tolower);
        if (hay.find(needle) == std::string::npos)
          continue;
      }

      char addrBuf[64] = "";
      if (g_TraceDisplayAddress)
      {
        snprintf(addrBuf, sizeof(addrBuf), "0x%llX", (unsigned long long)state.rva);
      }

      std::string nameToShow;
      if (g_TraceDisplayClassName)
      {
        nameToShow = state.className + "::" + state.methodName;
      }
      else
      {
        nameToShow = state.methodName;
      }

      char rowLabel[1024];
      snprintf(rowLabel, sizeof(rowLabel), "(%dx) (%s) %s", state.hitCount, addrBuf, nameToShow.c_str());

      ImGui::PushID(ptr);

      ImGui::TextUnformatted(rowLabel);
      float disableW = ImGui::CalcTextSize("Disable").x + ImGui::GetStyle().FramePadding.x * 2.0f;
      float enableW = ImGui::CalcTextSize("Enable").x + ImGui::GetStyle().FramePadding.x * 2.0f;
      float btnW1 = std::max(65.0f * g_FontScale, state.enabled ? disableW : enableW);
      float btnW2 = std::max(65.0f * g_FontScale, ImGui::CalcTextSize("Restore").x + ImGui::GetStyle().FramePadding.x * 2.0f);
      float totalBtnW = btnW1 + btnW2 + ImGui::GetStyle().ItemSpacing.x;

      ImGui::SameLine(std::max(0.0f, availW - totalBtnW));

      if (ImGui::Button(state.enabled ? "Disable" : "Enable", ImVec2(btnW1, 0.0f)))
      {
        std::lock_guard<std::mutex> lock(g_TraceMutex);
        g_TracedMethodsMap[ptr].enabled = !g_TracedMethodsMap[ptr].enabled;
      }
      ImGui::SameLine();
      if (ImGui::Button("Restore", ImVec2(btnW2, 0.0f)))
      {
        toRestore.push_back(ptr);
      }

      ImGui::PopID();
    }

    // Handle deferred restoration
    if (!toRestore.empty())
    {
      std::lock_guard<std::mutex> lock(g_TraceMutex);
      for (void *ptr : toRestore)
      {
        DobbyDestroy(ptr);
        g_TracedMethodsMap.erase(ptr);
        auto orderIt = std::find(g_TracedMethodsOrder.begin(), g_TracedMethodsOrder.end(), ptr);
        if (orderIt != g_TracedMethodsOrder.end())
        {
          g_TracedMethodsOrder.erase(orderIt);
        }
      }
    }

    ImGui::EndChild();

    // Bottom Controls
    ImGui::Checkbox("Sort by Hit Count", &g_TraceSortByHit);
    ImGui::SameLine();

    ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.25f, 0.43f, 1.00f));
    if (ImGui::Button("Restore All"))
    {
      std::lock_guard<std::mutex> lock(g_TraceMutex);
      for (void *ptr : g_TracedMethodsOrder)
      {
        DobbyDestroy(ptr);
      }
      g_TracedMethodsMap.clear();
      g_TracedMethodsOrder.clear();
    }
    ImGui::SameLine();
    if (ImGui::Button("Restore Any Hit"))
    {
      std::lock_guard<std::mutex> lock(g_TraceMutex);
      for (auto it = g_TracedMethodsMap.begin(); it != g_TracedMethodsMap.end();)
      {
        if (it->second.hitCount > 0)
        {
          DobbyDestroy(it->first);
          auto orderIt = std::find(g_TracedMethodsOrder.begin(), g_TracedMethodsOrder.end(), it->first);
          if (orderIt != g_TracedMethodsOrder.end())
          {
            g_TracedMethodsOrder.erase(orderIt);
          }
          it = g_TracedMethodsMap.erase(it);
        }
        else
        {
          ++it;
        }
      }
    }
    ImGui::SameLine();
    if (ImGui::Button("Restore No Hit"))
    {
      std::lock_guard<std::mutex> lock(g_TraceMutex);
      for (auto it = g_TracedMethodsMap.begin(); it != g_TracedMethodsMap.end();)
      {
        if (it->second.hitCount == 0)
        {
          DobbyDestroy(it->first);
          auto orderIt = std::find(g_TracedMethodsOrder.begin(), g_TracedMethodsOrder.end(), it->first);
          if (orderIt != g_TracedMethodsOrder.end())
          {
            g_TracedMethodsOrder.erase(orderIt);
          }
          it = g_TracedMethodsMap.erase(it);
        }
        else
        {
          ++it;
        }
      }
    }
    ImGui::PopStyleColor();
  }
  else
  {
    snprintf(sepText, sizeof(sepText), "Trace Logs (x%d)", (int)g_TraceLogs.size());
    ImGui::SeparatorText(sepText);

    ImVec2 listSize(0.0f, ImGui::GetContentRegionAvail().y - 45.0f);
    ImGui::BeginChild("##TraceLogsList", listSize, true);

    g_TraceMutex.lock();
    auto logsCopy = g_TraceLogs;
    g_TraceMutex.unlock();

    for (size_t i = 0; i < logsCopy.size(); ++i)
    {
      const auto &entry = logsCopy[i];

      std::hash<std::string> hasher;
      size_t hash = hasher(entry.fullName);
      ImVec4 color;
      switch (hash % 6)
      {
      case 0:
        color = ImVec4(0.3f, 0.6f, 1.0f, 1.0f);
        break; // Blue
      case 1:
        color = ImVec4(0.3f, 0.8f, 0.3f, 1.0f);
        break; // Green
      case 2:
        color = ImVec4(1.0f, 0.7f, 0.2f, 1.0f);
        break; // Yellow/Orange
      case 3:
        color = ImVec4(0.2f, 0.8f, 0.8f, 1.0f);
        break; // Cyan
      case 4:
        color = ImVec4(1.0f, 0.3f, 0.3f, 1.0f);
        break; // Red
      case 5:
        color = ImVec4(0.8f, 0.4f, 0.9f, 1.0f);
        break; // Purple
      }

      std::string text;
      if (g_TraceDisplayClassName)
      {
        text = entry.className + "::" + entry.methodName;
      }
      else
      {
        text = entry.methodName;
      }

      ImGui::TextColored(color, "%s", text.c_str());
    }

    ImGui::EndChild();

    ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.25f, 0.43f, 1.00f));
    if (ImGui::Button("Clear"))
    {
      std::lock_guard<std::mutex> lock(g_TraceMutex);
      g_TraceLogs.clear();
    }
    ImGui::PopStyleColor();
  }
}

void BeginDraw()
{ // Menu
  if (g_ShowMenuToggler)
  {
    const float logoSize = 132.0f;
    ImGui::SetNextWindowPos(ImVec2(28.0f, 160.0f), ImGuiCond_FirstUseEver);
    ImGui::SetNextWindowSize(ImVec2(logoSize, logoSize), ImGuiCond_Always);
    ImGuiWindowFlags logoFlags = ImGuiWindowFlags_NoDecoration |
                                 ImGuiWindowFlags_NoSavedSettings |
                                 ImGuiWindowFlags_NoScrollbar |
                                 ImGuiWindowFlags_NoBackground;
    if (ImGui::Begin("##OverlayLogo", nullptr, logoFlags))
    {
      ImVec2 logoPos = ImGui::GetWindowPos();
      ImVec2 logoSizeVec = ImGui::GetWindowSize();
      UpdateOverlayLogoBounds(logoPos.x, logoPos.y, logoSizeVec.x, logoSizeVec.y);
      ImDrawList *draw = ImGui::GetWindowDrawList();
      ImVec2 center = ImVec2(logoPos.x + logoSizeVec.x * 0.5f, logoPos.y + logoSizeVec.y * 0.5f);
      float radius = (logoSizeVec.x < logoSizeVec.y ? logoSizeVec.x : logoSizeVec.y) * 0.42f;
      draw->AddCircleFilled(center, radius, IM_COL32(70, 30, 105, 225), 48);
      draw->AddCircle(center, radius, IM_COL32(230, 220, 255, 240), 48, 3.0f);
      if (EnsureOverlayLogoTexture())
      {
        ImVec2 imageMin = ImVec2(center.x - radius + 4.0f, center.y - radius + 4.0f);
        ImVec2 imageMax = ImVec2(center.x + radius - 4.0f, center.y + radius - 4.0f);
        draw->AddImageRounded(GetOverlayLogoTexture(), imageMin, imageMax, ImVec2(0.0f, 0.0f), ImVec2(1.0f, 1.0f), IM_COL32_WHITE, radius - 4.0f);
      }
      else
      {
        draw->AddText(ImVec2(center.x - 20.0f, center.y - 20.0f), IM_COL32(255, 255, 255, 255), "V7");
      }
    }
    ImGui::End();
  }
  else
  {
    UpdateOverlayLogoBounds(0.0f, 0.0f, 0.0f, 0.0f);
  }

  DrawTraceOverlay();

  if (!IsOverlayMenuVisible())
  {
    UpdateOverlayMenuBounds(0.0f, 0.0f, 0.0f, 0.0f);
    return;
  }

  LoadConfig();
  ImVec2 center = ImGui::GetMainViewport()->GetCenter();
  ImVec2 workSize = ImGui::GetMainViewport()->WorkSize;
  ImVec2 defaultMenuSize(
      std::min(560.0f * g_FontScale, workSize.x * 0.92f),
      std::min(820.0f * g_FontScale, workSize.y * 0.88f));
  ImGui::SetNextWindowPos(center, ImGuiCond_Appearing, ImVec2(0.5f, 0.5f));
  ImGui::SetNextWindowSize(defaultMenuSize, ImGuiCond_FirstUseEver);
  if (g_SetWindowSettingsFromConfig && g_MenuPosSavedX >= 0.0f)
  {
    ImGui::SetNextWindowPos(ImVec2(g_MenuPosSavedX, g_MenuPosSavedY), ImGuiCond_Always);
    ImGui::SetNextWindowSize(ImVec2(g_MenuSizeSavedW, g_MenuSizeSavedH), ImGuiCond_Always);
    ImGui::SetNextWindowCollapsed(g_MenuCollapsedSaved, ImGuiCond_Always);
    g_SetWindowSettingsFromConfig = false;
  }
  ImGuiWindowFlags menuFlags = ImGuiWindowFlags_NoSavedSettings;
  ImGui::PushStyleVar(ImGuiStyleVar_WindowPadding, ImVec2(12.0f, 10.0f));
  ImGui::PushStyleVar(ImGuiStyleVar_FramePadding, ImVec2(8.0f, 6.0f));
  ImGui::PushStyleVar(ImGuiStyleVar_ItemSpacing, ImVec2(6.0f, 6.0f));
  ImGui::PushStyleVar(ImGuiStyleVar_CellPadding, ImVec2(6.0f, 4.0f));
  ImGui::PushStyleVar(ImGuiStyleVar_WindowRounding, 8.0f);
  ImGui::PushStyleVar(ImGuiStyleVar_FrameRounding, 5.0f);
  ImGui::PushStyleVar(ImGuiStyleVar_TabRounding, 5.0f);
  ImGui::PushStyleColor(ImGuiCol_WindowBg, ImVec4(0.06f, 0.06f, 0.09f, 0.96f));
  ImGui::PushStyleColor(ImGuiCol_ChildBg, ImVec4(0.06f, 0.06f, 0.09f, 0.76f));
  ImGui::PushStyleColor(ImGuiCol_Border, ImVec4(0.25f, 0.32f, 0.55f, 0.80f));
  ImGui::PushStyleColor(ImGuiCol_FrameBg, ImVec4(0.09f, 0.11f, 0.17f, 0.92f));
  ImGui::PushStyleColor(ImGuiCol_FrameBgHovered, ImVec4(0.14f, 0.17f, 0.26f, 0.96f));
  ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.25f, 0.43f, 0.92f));
  ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.28f, 0.35f, 0.60f, 0.98f));
  ImGui::PushStyleColor(ImGuiCol_ButtonActive, ImVec4(0.35f, 0.44f, 0.75f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_Tab, ImVec4(0.11f, 0.13f, 0.21f, 0.95f));
  ImGui::PushStyleColor(ImGuiCol_TabHovered, ImVec4(0.20f, 0.25f, 0.40f, 0.98f));
  ImGui::PushStyleColor(ImGuiCol_TabSelected, ImVec4(0.29f, 0.37f, 0.65f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_Header, ImVec4(0.20f, 0.25f, 0.43f, 0.85f));
  ImGui::PushStyleColor(ImGuiCol_HeaderHovered, ImVec4(0.28f, 0.35f, 0.60f, 0.92f));
  ImGui::PushStyleColor(ImGuiCol_HeaderActive, ImVec4(0.35f, 0.44f, 0.75f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_CheckMark, ImVec4(0.29f, 0.37f, 0.65f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_SliderGrab, ImVec4(0.29f, 0.37f, 0.65f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_SliderGrabActive, ImVec4(0.35f, 0.44f, 0.75f, 1.00f));
  ImGui::PushStyleColor(ImGuiCol_Separator, ImVec4(0.25f, 0.32f, 0.55f, 0.50f));
  ImGui::PushStyleColor(ImGuiCol_SeparatorHovered, ImVec4(0.29f, 0.37f, 0.65f, 0.70f));
  ImGui::PushStyleColor(ImGuiCol_SeparatorActive, ImVec4(0.35f, 0.44f, 0.75f, 1.00f));
  bool menuOpen = ImGui::Begin("Modder By SilverV7", 0, menuFlags);
  ImVec2 pos = ImGui::GetWindowPos();
  ImVec2 size = ImGui::GetWindowSize();
  UpdateOverlayMenuBounds(pos.x, pos.y, size.x, size.y);

  g_MenuPosSavedX = pos.x;
  g_MenuPosSavedY = pos.y;
  g_MenuSizeSavedW = size.x;
  g_MenuSizeSavedH = size.y;
  g_MenuCollapsedSaved = ImGui::IsWindowCollapsed();

  if (menuOpen)
  {
    // Popups are opened directly by the triggering buttons

    LoadConfig();
    static double s_NextMetadataAutoLoadTime = 0.0;
    double now = ImGui::GetTime();
    if (g_KeyUnlocked && !g_MetadataLoaded && now >= s_NextMetadataAutoLoadTime)
    {
      RefreshMetadataCache();
      if (!g_MetadataLoaded)
      {
        s_NextMetadataAutoLoadTime = now + 2.0;
      }
    }
    CapturePendingCallerResult();
    // DrawMenuHeader();

    if (ImGui::BeginTabBar("##MainTabs"))
    {
      if (!g_KeyUnlocked)
      {
        if (ImGui::BeginTabItem("Login"))
        {
          RenderKeyGate();
          ImGui::EndTabItem();
        }
      }
      else
      {
        static bool firstRun = true;
        ImGuiTabItemFlags toolsTabFlags = (g_RequestToolsTab || firstRun) ? ImGuiTabItemFlags_SetSelected : 0;
        firstRun = false;

        if (ImGui::BeginTabItem("Main"))
        {
          g_ActiveTabIndex = 0;

#if 0
#endif

        if (ImGui::Checkbox("Speed", &g_SpeedEnabled))
        {
          SpeedHack::SetEnabled(g_SpeedEnabled);
        }
        ImGui::SameLine();
        ImGui::PushItemWidth(260.0f);
        if (ImGui::SliderFloat("##SpeedScale", &g_TimeScaleValue, 0.1f, 20.0f, "%.1fx"))
        {
          if (g_SpeedEnabled)
            SpeedHack::Apply(g_TimeScaleValue);
        }
        ImGui::PopItemWidth();

        if (ImGui::Checkbox("ALL Level ", &HookAllLevel))
        {
          if (HookAllLevel)
          {
            SymbolLevel::SetEnabled(true);
            SkillTreeLevel::SetEnabled(true);
            PvpLevel::SetEnabled(true);
          }
        }

        ImGui::Text(" Symbol:");
        ImGui::SameLine();
        ImGui::PushItemWidth(200.0f);
        ImGui::InputInt("##SymbolLevelValue", &SymbolLevelValue, 1, 100, ImGuiInputTextFlags_CharsDecimal);
        ImGui::PopItemWidth();

        ImGui::Text("Skill:");
        ImGui::SameLine();
        ImGui::PushItemWidth(200.0f);
        ImGui::InputInt("##SkillTreeLevelValue", &SkillTreeLevelValue, 1, 100, ImGuiInputTextFlags_CharsDecimal);
        ImGui::PopItemWidth();

        ImGui::Text(" PvP:");
        ImGui::SameLine();
        ImGui::PushItemWidth(200.0f);
        ImGui::InputInt("##PvpLevelValue", &PvpLevelValue, 1, 100, ImGuiInputTextFlags_CharsDecimal);
        ImGui::PopItemWidth();

        ImGui::Spacing();
        ImGui::Separator();
        if (ImGui::Checkbox("ALL IN ONE", &g_AllInOneEnabled))
        {
          if (g_AllInOneEnabled)
          {
            static const char *kAllInOneScript = R"lua(
local function safe(name, fn)
    local ok, err = pcall(fn)
    if not ok then loge(name .. " error: " .. tostring(err)) end
end

local function objs(name)
    local c = Class.fromName(name)
    if not c then
        loge(name .. " class nil")
        return {}
    end

    local o = c:findObjects() or {}
    logi(name .. " found = " .. tostring(#o))
    return o
end

for _, v in ipairs(objs("DarkIdle2.CharacterData")) do
    safe("CharacterData", function()
        v.level = 1500
    end)
end

for _, obj in ipairs(objs("DarkIdle2.AwakenManager")) do
    safe("AwakenManager", function()
        local d = obj._awakenLevel
        if d then
            d:dictSetAllInt(200)
        end
    end)

    safe("EngraveData from AwakenManager", function()
        local engrave = obj._engraveData
        if engrave then
            local list = engrave.EngraveLevelList
            if list then
                list["set_Item"](list, 0, 10000)
                list["set_Item"](list, 1, 10000)

                for i = 2, 19 do
                    list["set_Item"](list, i, 1000)
                end
            end
        end
    end)
end

for _, v in ipairs(objs("DarkIdle2.EngraveData")) do
    safe("EngraveData direct", function()
        local list = v.EngraveLevelList
        if list then
            list["set_Item"](list, 0, 10000)
            list["set_Item"](list, 1, 10000)

            for i = 2, 19 do
                list["set_Item"](list, i, 1000)
            end
        end
    end)
end

for _, obj in ipairs(objs("DarkIdle2.StarlightBlessData")) do
    safe("StarlightBlessData", function()
        obj["set_BuffCount"](obj, 2000000000)
    end)
end

for _, obj in ipairs(objs("DarkIdle2.UpgradeData")) do
    safe("UpgradeData", function()
        local d = obj.level
        if d then
            d:dictSetAllInt(100000)
        end
    end)
end

for _, obj in ipairs(objs("DarkIdle2.Artifact.ArtifactSaveData")) do
    safe("ArtifactSaveData", function()
        obj["set_IsActive"](obj, true)
        obj["set_Grade"](obj, 20)
    end)
end
)lua";
            LuaConsole_RequestExecute(kAllInOneScript);
          }
        }

          if (ImGui::Checkbox("Free Shop", &g_PetLevelEnabled))
          {
            if (g_PetLevelEnabled)
            {
              static const char *kPetLevelScript = R"lua(
Hook.returnBool("ShopItemInfo", "get_IsCash", false)
)lua";
              LuaConsole_RequestExecute(kPetLevelScript, "FreeShop");
            }
            else
            {
              LuaConsole_RestoreTransaction("FreeShop");
            }
          }

#if 0
        if (ImGui::Checkbox("Stat", &g_StatEnabled))
        {
          if (g_StatEnabled)
          {
            static const char *kStatScript = R"lua(
local function safe(name, fn)
    local ok, err = pcall(fn)
    if not ok then loge(name .. " error: " .. tostring(err)) end
end

local c = Class.fromName("DarkIdle2.UpgradeData")
local os = c and c:findObjects() or {}

local n = 0

local function setKeys(list, keys)
    if not list then return end

    for i = 0, 3 do
        local key = keys[i]
        if key ~= nil then
            list:listSetStructFieldInt(i, "key", key)
            n = n + 1
        end
    end
end

for _, obj in ipairs(os) do
    safe("slotOptions", function()
        local dict = obj.slotOptions
        if not dict then return end

        setKeys(dict:dictGetValueObjectByRawIndex(0), {
            [0] = 7, [1] = 42, [2] = 21, [3] = 21
        })

        local entryKeys = {
            [1] = 142,
            [2] = 549,
            [3] = 556,
            [4] = 428,
            [5] = 507
        }

        for rawIndex, key in pairs(entryKeys) do
            setKeys(dict:dictGetValueObjectByRawIndex(rawIndex), {
                [0] = key, [1] = key, [2] = key, [3] = key
            })
        end
    end)
end

print("Done: " .. tostring(n))
)lua";
            LuaConsole_RequestExecute(kStatScript);
          }
        }
#endif

          // Custom menu items (created via "Create Checkbox Button" in Script tab)
          // Capture pending delete outside loop to avoid PushID scope issue with OpenPopup
          int pendingDeleteIndex = -1;
          for (size_t i = 0; i < g_CustomMenuItems.size(); ++i)
          {
            auto &item = g_CustomMenuItems[i];
            bool wasEnabled = item.enabled;
            if (ImGui::Checkbox(item.name.c_str(), &item.enabled))
            {
              if (item.enabled && !wasEnabled)
                LuaConsole_RequestExecute(item.script.c_str(), item.name.c_str());
              else if (!item.enabled && wasEnabled)
                LuaConsole_RestoreTransaction(item.name.c_str());
            }
            ImGui::SameLine();
            ImGui::PushID((int)(1000 + i));
            if (ImGui::SmallButton("E"))
            {
              g_EditingIndex = (int)i;
              snprintf(g_EditNameBuf, sizeof(g_EditNameBuf), "%s", item.name.c_str());
              LuaConsole_SetScriptText(item.script.c_str());
            }
            ImGui::SameLine();
            if (ImGui::SmallButton("X"))
              pendingDeleteIndex = (int)i; // mark; call OpenPopup AFTER PopID
            ImGui::PopID();
          }

          if (g_ShowSpeedControl)
          {
            if (ImGui::Checkbox("Speed", &g_SpeedEnabled))
            {
              SpeedHack::SetEnabled(g_SpeedEnabled);
            }
            ImGui::SameLine();
            ImGui::PushItemWidth(260.0f);
            if (ImGui::SliderFloat("##SpeedScale", &g_TimeScaleValue, 1.0f, 10.0f, "%.1fx"))
            {
              if (g_SpeedEnabled)
                SpeedHack::Apply(g_TimeScaleValue);
            }
            ImGui::PopItemWidth();
            ImGui::SameLine();
            if (ImGui::SmallButton("X##Speed"))
            {
              g_ShowSpeedControl = false;
              SaveConfig();
            }
          }

#if 0 // ── Add Ruby Trigger ─────────────────────────────────────────────
        ImGui::Spacing();
        ImGui::Separator();
        ImGui::TextUnformatted("Add Ruby");
        ImGui::PushItemWidth(200.0f);
        ImGui::InputInt("##RubyCount", &g_RubyCount, 1, 100, ImGuiInputTextFlags_CharsDecimal);
        ImGui::PopItemWidth();
        ImGui::SameLine();
        if (ImGui::Button("Trigger", ImVec2(100, 0)))
        {
          RubyAdd::Trigger(g_RubyCount);
        }
#endif
          // Open delete popup outside PushID scope so the ID resolves correctly
          if (pendingDeleteIndex >= 0)
          {
            g_DeleteConfirmIndex = pendingDeleteIndex;
            ImGui::OpenPopup("##DeleteConfirm");
          }

          // Delete confirmation popup
          if (ImGui::BeginPopupModal("##DeleteConfirm", nullptr, ImGuiWindowFlags_AlwaysAutoResize))
          {
            ImGui::Text("Delete  \"%s\" ?",
                        (g_DeleteConfirmIndex >= 0 && g_DeleteConfirmIndex < (int)g_CustomMenuItems.size())
                            ? g_CustomMenuItems[g_DeleteConfirmIndex].name.c_str()
                            : "?");
            ImGui::Separator();
            if (ImGui::Button("Yes, Delete", ImVec2(120, 0)))
            {
              if (g_DeleteConfirmIndex >= 0 && g_DeleteConfirmIndex < (int)g_CustomMenuItems.size())
              {
                if (g_CustomMenuItems[g_DeleteConfirmIndex].enabled)
                {
                  LuaConsole_RestoreTransaction(g_CustomMenuItems[g_DeleteConfirmIndex].name.c_str());
                }
                // If currently editing this item, cancel edit too
                if (g_EditingIndex == g_DeleteConfirmIndex)
                  g_EditingIndex = -1;
                else if (g_EditingIndex > g_DeleteConfirmIndex)
                  g_EditingIndex--; // shift index down
                g_CustomMenuItems.erase(g_CustomMenuItems.begin() + g_DeleteConfirmIndex);
                SaveConfig();
              }
              g_DeleteConfirmIndex = -1;
              ImGui::CloseCurrentPopup();
            }
            ImGui::SameLine();
            if (ImGui::Button("Cancel", ImVec2(100, 0)))
            {
              g_DeleteConfirmIndex = -1;
              ImGui::CloseCurrentPopup();
            }
            ImGui::EndPopup();
          }
          ImGui::EndTabItem();
        }

        if (!g_SilentMode && g_ShowTabScript && ImGui::BeginTabItem("Script"))
        {
          g_ActiveTabIndex = 1;

          // Edit mode: show name input + Save / Delete / Cancel buttons at top
          if (g_EditingIndex >= 0 && g_EditingIndex < (int)g_CustomMenuItems.size())
          {
            // ── Edit-mode header: name input + responsive 3-button row ─────
            ImGui::TextUnformatted("Editing:");
            float editAvail = ImGui::GetContentRegionAvail().x;
            float editSp = ImGui::GetStyle().ItemSpacing.x;
            // Reserve space for 3 equal buttons (Save / Delete / Cancel)
            float editBtnW = std::max(60.0f, (editAvail - editSp * 2.0f) / 3.0f);
            ImGui::SetNextItemWidth(std::max(100.0f, editAvail - editBtnW * 3.0f - editSp * 3.0f));
            ImGui::InputText("##editname", g_EditNameBuf, sizeof(g_EditNameBuf));
            ImGui::SameLine();
            if (ImGui::Button("Save", ImVec2(editBtnW, 0)))
            {
              if (g_EditNameBuf[0])
              {
                auto &editItem = g_CustomMenuItems[g_EditingIndex];
                editItem.name = g_EditNameBuf;
                editItem.script = LuaConsole_GetScriptText();
                SaveConfig();
              }
              g_EditingIndex = -1;
            }
            ImGui::SameLine();
            ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.55f, 0.10f, 0.10f, 0.70f));
            ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.75f, 0.15f, 0.15f, 0.90f));
            ImGui::PushStyleColor(ImGuiCol_ButtonActive, ImVec4(0.90f, 0.20f, 0.20f, 1.00f));
            if (ImGui::Button("Delete", ImVec2(editBtnW, 0)))
            {
              g_DeleteConfirmIndex = g_EditingIndex;
              ImGui::OpenPopup("##DeleteConfirmEdit");
            }
            ImGui::PopStyleColor(3);
            ImGui::SameLine();
            if (ImGui::Button("Cancel", ImVec2(editBtnW, 0)))
              g_EditingIndex = -1;

            // Delete confirmation popup (Script tab)
            if (ImGui::BeginPopupModal("##DeleteConfirmEdit", nullptr, ImGuiWindowFlags_AlwaysAutoResize))
            {
              ImGui::Text("Delete  \"%s\" ?",
                          (g_DeleteConfirmIndex >= 0 && g_DeleteConfirmIndex < (int)g_CustomMenuItems.size())
                              ? g_CustomMenuItems[g_DeleteConfirmIndex].name.c_str()
                              : "?");
              ImGui::Separator();
              if (ImGui::Button("Yes, Delete", ImVec2(120, 0)))
              {
                if (g_DeleteConfirmIndex >= 0 && g_DeleteConfirmIndex < (int)g_CustomMenuItems.size())
                {
                  if (g_CustomMenuItems[g_DeleteConfirmIndex].enabled)
                  {
                    LuaConsole_RestoreTransaction(g_CustomMenuItems[g_DeleteConfirmIndex].name.c_str());
                  }
                  g_CustomMenuItems.erase(g_CustomMenuItems.begin() + g_DeleteConfirmIndex);
                }
                SaveConfig();
                g_DeleteConfirmIndex = -1;
                g_EditingIndex = -1;
                ImGui::CloseCurrentPopup();
              }
              ImGui::SameLine();
              if (ImGui::Button("Cancel", ImVec2(100, 0)))
              {
                g_DeleteConfirmIndex = -1;
                ImGui::CloseCurrentPopup();
              }
              ImGui::EndPopup();
            }

            ImGui::Separator();
          }

          LuaConsole_Draw();

          if (LuaConsole_IsFileOpened())
          {
            ImGui::Spacing();
            ImGui::Separator();
            ImGui::TextUnformatted("Persistence");
            {
              // Row 1: Save Config | Load Config  (split equally)
              float pAvail = ImGui::GetContentRegionAvail().x;
              float pSp = ImGui::GetStyle().ItemSpacing.x;
              float pW = (pAvail - pSp) * 0.5f;
              if (ImGui::Button("Save Config", ImVec2(pW, 0)))
                SaveConfig();
              ImGui::SameLine();
              if (ImGui::Button("Load Config", ImVec2(pW, 0)))
              {
                g_ConfigLoaded = false;
                LoadConfig();
              }
              // Row 2: Auto Save checkbox (standalone)
              if (ImGui::Checkbox("Auto Save", &g_AutoSaveScript))
                SaveConfig();
            }

            ImGui::Spacing();
            if (ImGui::Button("Create Checkbox Button",
                              ImVec2(ImGui::GetContentRegionAvail().x, 0)))
            {
              const char *script = LuaConsole_GetScriptText();
              if (!script || !script[0])
              {
                // Script is empty – show error, do NOT open popup
                snprintf(g_CreateErrorMsg, sizeof(g_CreateErrorMsg),
                         "Script is empty. Write a script first.");
              }
              else
              {
                g_NewItemName[0] = '\0';
                g_CreateErrorMsg[0] = '\0';
                g_ShowCreatePopup = true;
                ImGui::OpenPopup("##CreateBtnPopup");
              }
            }

            // Inline error shown below the button (script-empty case)
            if (g_CreateErrorMsg[0])
            {
              ImGui::PushStyleColor(ImGuiCol_Text, ImVec4(1.0f, 0.35f, 0.35f, 1.0f));
              ImGui::TextUnformatted(g_CreateErrorMsg);
              ImGui::PopStyleColor();
            }

            ImGui::Spacing();
            if (ImGui::Button("Add Btn Speed", ImVec2(ImGui::GetContentRegionAvail().x, 0)))
            {
              g_ShowSpeedControl = true;
              SaveConfig();
            }

            if (ImGui::BeginPopupModal("##CreateBtnPopup", &g_ShowCreatePopup, ImGuiWindowFlags_AlwaysAutoResize))
            {
              ImGui::TextUnformatted("Button name:");
              bool hitEnter = ImGui::InputText("##newitemname", g_NewItemName,
                                               sizeof(g_NewItemName), ImGuiInputTextFlags_EnterReturnsTrue);

              // Inline error inside popup (empty name / duplicate)
              if (g_CreateErrorMsg[0])
              {
                ImGui::PushStyleColor(ImGuiCol_Text, ImVec4(1.0f, 0.35f, 0.35f, 1.0f));
                ImGui::TextUnformatted(g_CreateErrorMsg);
                ImGui::PopStyleColor();
              }

              ImGui::Spacing();
              bool doCreate = hitEnter || ImGui::Button("Create", ImVec2(120, 0));
              if (doCreate)
              {
                if (!g_NewItemName[0])
                {
                  snprintf(g_CreateErrorMsg, sizeof(g_CreateErrorMsg),
                           "Name cannot be empty.");
                }
                else
                {
                  bool duplicate = false;
                  for (auto &item : g_CustomMenuItems)
                  {
                    if (item.name == g_NewItemName)
                    {
                      duplicate = true;
                      break;
                    }
                  }
                  if (duplicate)
                  {
                    snprintf(g_CreateErrorMsg, sizeof(g_CreateErrorMsg),
                             "'%s' already exists.", g_NewItemName);
                  }
                  else
                  {
                    CustomMenuItem newItem;
                    newItem.name = g_NewItemName;
                    newItem.script = LuaConsole_GetScriptText();
                    newItem.enabled = false;
                    g_CustomMenuItems.push_back(newItem);
                    SaveConfig();
                    g_CreateErrorMsg[0] = '\0';
                    g_ShowCreatePopup = false;
                    ImGui::CloseCurrentPopup();
                  }
                }
              }
              ImGui::SameLine();
              if (ImGui::Button("Cancel", ImVec2(120, 0)))
              {
                g_CreateErrorMsg[0] = '\0';
                g_ShowCreatePopup = false;
                ImGui::CloseCurrentPopup();
              }
              ImGui::EndPopup();
            }
          }

          ImGui::EndTabItem();
        }

        // ── Patched Tab ──────────────────────────────────────────────────────────
        if (!g_SilentMode && g_ShowTabPatched && ImGui::BeginTabItem("Patched"))
        {
          g_ActiveTabIndex = 4;

          std::vector<PatchEntry> patches = LuaConsole_GetActivePatchList();
          std::vector<FieldPatchEntry> fields = LuaConsole_GetModifiedFields();

          // Single scrollable child for the entire Patched tab content
          ImGui::PushStyleVar(ImGuiStyleVar_ItemSpacing, ImVec2(4, 3));
          ImGui::PushStyleVar(ImGuiStyleVar_FramePadding, ImVec2(6, 2));

          ImGui::BeginChild("##PatchedScroll", ImVec2(0, 0), false,
                            ImGuiWindowFlags_AlwaysUseWindowPadding);

          // ── SECTION 1: Active Patches ─────────────────────────────────────────
          int activeCount = 0;
          for (auto &p : patches)
            if (p.enabled)
              ++activeCount;

          ImGui::PushStyleColor(ImGuiCol_Text, ImVec4(0.75f, 0.75f, 0.80f, 1.00f));
          ImGui::TextUnformatted("Active Patches");
          ImGui::PopStyleColor();
          ImGui::SameLine();
          ImGui::TextDisabled("(%d)", activeCount);

          if (!patches.empty())
          {
            float daW = ImGui::CalcTextSize("Disable All").x +
                        ImGui::GetStyle().FramePadding.x * 2.0f;
            ImGui::SameLine(
                ImGui::GetWindowPos().x - ImGui::GetCursorPosX() +
                ImGui::GetContentRegionMax().x - daW);
            if (ImGui::SmallButton("Disable All"))
            {
              DisableAllRedirectInt1();
              for (auto &p : patches)
              {
                if (p.type != "Int1" && p.enabled)
                  LuaConsole_PatchToggleEnable(p.nativeKey, p.type);
              }
            }
          }

          ImGui::Separator();
          ImGui::Spacing();

          static const ImVec4 kCardBgEven(0.13f, 0.09f, 0.19f, 0.55f);
          static const ImVec4 kCardBgOdd(0.09f, 0.06f, 0.13f, 0.55f);
          float cardW = ImGui::GetContentRegionAvail().x;

          if (patches.empty())
          {
            ImGui::TextDisabled("No active patches");
          }
          else
          {
            for (int i = 0; i < (int)patches.size(); ++i)
            {
              PatchEntry &pe = patches[i];
              ImGui::PushID(i);

              ImVec2 cardMin = ImGui::GetCursorScreenPos();
              ImGui::BeginGroup();

              ImGui::TextDisabled("[%s]", pe.type.c_str());
              ImGui::SameLine(0, 6);
              char route[192];
              if (pe.type.rfind("Return", 0) == 0)
              {
                snprintf(route, sizeof(route), "%s.%s = %s",
                         pe.sourceClass.c_str(), pe.sourceMethod.c_str(),
                         pe.targetMethod.c_str());
              }
              else
              {
                snprintf(route, sizeof(route), "%s.%s -> %s.%s",
                         pe.sourceClass.c_str(), pe.sourceMethod.c_str(),
                         pe.targetClass.c_str(), pe.targetMethod.c_str());
              }
              ImGui::TextUnformatted(route);

              ImGui::Indent(4.0f);
              if (pe.enabled)
              {
                ImGui::PushStyleColor(ImGuiCol_Text, ImVec4(0.25f, 0.90f, 0.45f, 1.00f));
                ImGui::TextUnformatted("ACTIVE");
                ImGui::PopStyleColor();
              }
              else
              {
                ImGui::PushStyleColor(ImGuiCol_Text, ImVec4(0.50f, 0.50f, 0.55f, 1.00f));
                ImGui::TextUnformatted("DISABLED");
                ImGui::PopStyleColor();
              }
              ImGui::Unindent(4.0f);

              ImGui::Indent(4.0f);
              const char *toggleLabel = pe.enabled ? "Disable" : "Enable";
              if (ImGui::SmallButton(toggleLabel))
                LuaConsole_PatchToggleEnable(pe.nativeKey, pe.type);
              ImGui::SameLine(0, 6);
              if (ImGui::SmallButton("Return"))
                LuaConsole_PatchReturn(pe.nativeKey, pe.type);
              ImGui::Unindent(4.0f);

              ImGui::EndGroup();

              ImVec2 cardMax = ImGui::GetItemRectMax();
              cardMax.x = cardMin.x + cardW;
              ImVec4 bgColor = (i % 2 == 0) ? kCardBgEven : kCardBgOdd;
              ImGui::GetWindowDrawList()->AddRectFilled(
                  cardMin, cardMax,
                  ImGui::ColorConvertFloat4ToU32(bgColor), 4.0f);

              ImGui::Spacing();
              if (i < (int)patches.size() - 1)
              {
                ImGui::PushStyleColor(ImGuiCol_Separator, ImVec4(0.28f, 0.15f, 0.38f, 0.40f));
                ImGui::Separator();
                ImGui::PopStyleColor();
                ImGui::Spacing();
              }

              ImGui::PopID();
            }
          }

          ImGui::Spacing();
          ImGui::Spacing();

          // ── SECTION 2: Modified Fields ─────────────────────────────────────────
          ImGui::PushStyleColor(ImGuiCol_Text, ImVec4(0.75f, 0.75f, 0.80f, 1.00f));
          ImGui::TextUnformatted("Modified Fields");
          ImGui::PopStyleColor();
          ImGui::SameLine();
          ImGui::TextDisabled("(%d)", (int)fields.size());

          if (!fields.empty())
          {
            float raW = ImGui::CalcTextSize("Return All").x +
                        ImGui::GetStyle().FramePadding.x * 2.0f;
            ImGui::SameLine(
                ImGui::GetWindowPos().x - ImGui::GetCursorPosX() +
                ImGui::GetContentRegionMax().x - raW);
            if (ImGui::SmallButton("Return All"))
            {
              LuaConsole_RestoreAllFieldsManual();
            }
          }

          ImGui::Separator();
          ImGui::Spacing();

          if (fields.empty())
          {
            ImGui::TextDisabled("No modified fields");
          }
          else
          {
            for (int i = 0; i < (int)fields.size(); ++i)
            {
              FieldPatchEntry &fe = fields[i];
              ImGui::PushID(i + 20000); // Unique ID namespace for fields to avoid collision

              ImVec2 cardMin = ImGui::GetCursorScreenPos();
              ImGui::BeginGroup();

              // Line 1: [Field] Class.Field (Type)
              ImGui::TextDisabled("[Field]");
              ImGui::SameLine(0, 6);
              char fieldHeader[256];
              snprintf(fieldHeader, sizeof(fieldHeader), "%s.%s (%s)",
                       fe.className.c_str(), fe.fieldName.c_str(), fe.typeName.c_str());
              ImGui::TextUnformatted(fieldHeader);

              // Line 2: Original -> Current
              ImGui::Indent(4.0f);
              ImGui::Text("Value: %s -> %s", fe.origValue.c_str(), fe.currentValue.c_str());
              ImGui::Unindent(4.0f);

              // Line 3: Owner info
              ImGui::Indent(4.0f);
              ImGui::TextDisabled("Owner: %s", fe.menuName.c_str());
              ImGui::Unindent(4.0f);

              // Line 4: Return button
              ImGui::Indent(4.0f);
              if (ImGui::SmallButton("Return"))
              {
                LuaConsole_RestoreFieldManual(fe.obj, fe.offset);
              }
              ImGui::Unindent(4.0f);

              ImGui::EndGroup();

              ImVec2 cardMax = ImGui::GetItemRectMax();
              cardMax.x = cardMin.x + cardW;
              ImVec4 bgColor = (i % 2 == 0) ? kCardBgEven : kCardBgOdd;
              ImGui::GetWindowDrawList()->AddRectFilled(
                  cardMin, cardMax,
                  ImGui::ColorConvertFloat4ToU32(bgColor), 4.0f);

              ImGui::Spacing();
              if (i < (int)fields.size() - 1)
              {
                ImGui::PushStyleColor(ImGuiCol_Separator, ImVec4(0.28f, 0.15f, 0.38f, 0.40f));
                ImGui::Separator();
                ImGui::PopStyleColor();
                ImGui::Spacing();
              }

              ImGui::PopID();
            }
          }

          ImGui::EndChild();
          ImGui::PopStyleVar(2);
          ImGui::EndTabItem();
        }

        if (!g_SilentMode && g_ShowTabDump && ImGui::BeginTabItem("Dump"))
        {

          ImGui::Text("Select output directory to dump libil2cpp.so:");
          ImGui::Spacing();
          {
            // Two dump buttons: split available width equally
            float dAvail = ImGui::GetContentRegionAvail().x;
            float dSp = ImGui::GetStyle().ItemSpacing.x;
            float dW = (dAvail - dSp) * 0.5f;
            float dH = std::max(50.0f, dAvail * 0.07f); // proportional height
            if (ImGui::Button("Dump to Pictures", ImVec2(dW, dH)))
              StartDump("/sdcard/Pictures/");
            ImGui::SameLine();
            if (ImGui::Button("Dump to Downloads", ImVec2(dW, dH)))
              StartDump("/sdcard/Download/");
          }

          ImGui::Spacing();
          ImGui::Separator();
          ImGui::Spacing();

          ImGui::Text("Status: ");
          ImGui::SameLine();
          if (g_DumpStatus == 0)
          {
            ImGui::TextColored(ImVec4(0.7f, 0.7f, 0.7f, 1.0f), "Idle");
          }
          else if (g_DumpStatus == 1)
          {
            ImGui::TextColored(ImVec4(1.0f, 0.8f, 0.2f, 1.0f), "%s", g_DumpStatusMsg.c_str());
          }
          else if (g_DumpStatus == 2)
          {
            ImGui::TextColored(ImVec4(0.2f, 0.8f, 0.2f, 1.0f), "%s", g_DumpStatusMsg.c_str());
          }
          else if (g_DumpStatus == 3)
          {
            ImGui::TextColored(ImVec4(1.0f, 0.2f, 0.2f, 1.0f), "Failed!");
          }
          ImGui::EndTabItem();
        }

        if (!g_SilentMode && g_ShowTabTools && ImGui::BeginTabItem("Tools", nullptr, toolsTabFlags))
        {
          g_RequestToolsTab = false;
          g_ActiveTabIndex = 5;

          float availW = ImGui::GetContentRegionAvail().x;
          float itemSpacing = ImGui::GetStyle().ItemSpacing.x;

          // Render Inspector screen (Flat Layout only, when a class is selected)
          if (!g_SplitLayout && g_ToolsInspectorFullView && g_SelectedFlatIdx != -1 && g_SelectedFlatIdx < (int)g_FilteredList.size())
          {
            auto &selectedItem = g_FilteredList[g_SelectedFlatIdx];

            // Compact action row: right-aligned controls like the reference tool.
            float btnW = ImGui::GetFrameHeight();
            float compactBtnW = ImGui::CalcTextSize("Compact").x + ImGui::GetStyle().FramePadding.x * 2.0f;
            float traceAllW = ImGui::CalcTextSize("Trace All").x + ImGui::GetStyle().FramePadding.x * 2.0f;
            float restoreAllW = ImGui::CalcTextSize("Restore All").x + ImGui::GetStyle().FramePadding.x * 2.0f;
            float traceBtnW = std::max(traceAllW, restoreAllW);

            float rightControlsW = traceBtnW + compactBtnW + btnW + itemSpacing * 2.0f;
            float rightX = ImGui::GetWindowWidth() - ImGui::GetStyle().WindowPadding.x - rightControlsW;
            ImGui::SetCursorPosX(std::max(ImGui::GetCursorPosX(), rightX));

            ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.25f, 0.43f, 1.00f));
            ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.28f, 0.35f, 0.60f, 1.00f));
            bool classTraced = IsAnyMethodTraced(selectedItem);
            if (ImGui::Button(classTraced ? "Restore All" : "Trace All", ImVec2(traceBtnW, 0)))
            {
              if (classTraced)
                RestoreClassMethods(selectedItem);
              else
                TraceClassMethods(selectedItem);
            }

            ImGui::SameLine();
            RenderInspectorFullViewToggleButton("flat_full_view_exit", ImVec2(compactBtnW, 0));

            ImGui::SameLine();
            if (ImGui::Button("X", ImVec2(btnW, 0)))
            {
              g_SelectedFlatIdx = -1;
              g_ToolsInspectorFullView = false;
              SaveConfig();
            }
            ImGui::PopStyleColor(2);

            float filterLabelW = ImGui::CalcTextSize("Filter").x;
            float filteredW = ImGui::CalcTextSize("Filtered").x + ImGui::GetFrameHeight() + itemSpacing * 2.0f;
            float filterInputW = std::max(90.0f, availW - filterLabelW - filteredW - itemSpacing * 4.0f);
            ImGui::SetNextItemWidth(filterInputW);
            ImGui::InputText("##InspectorFilterInput", g_ObjectFieldFilter, sizeof(g_ObjectFieldFilter));
            ImGui::SameLine();
            ImGui::TextUnformatted("Filter");
            ImGui::SameLine();
            ImGui::Checkbox("Filtered", &g_InspectorFiltered);
            ImGui::Spacing();

            // 3. Collapsible sections content
            ImGui::BeginChild("##InspectorContent", ImVec2(0, 0), false, ImGuiWindowFlags_HorizontalScrollbar);
            ApplyDragScroll();

            RenderClassInspectorContent(selectedItem);

            ImGui::EndChild();
          }
          else
          {
            // Render search controls + list
            RenderSearchTabsBar();

            if (g_ActiveSearchTab >= 0 && g_ActiveSearchTab < (int)g_SearchTabs.size() &&
                (g_SearchTabs[g_ActiveSearchTab].type == SearchTabState::TYPE_INSPECT_OBJECT ||
                 g_SearchTabs[g_ActiveSearchTab].type == SearchTabState::TYPE_INSPECT_RAW))
            {
              auto &tab = g_SearchTabs[g_ActiveSearchTab];
              if (tab.inspectPtr)
              {
                auto &history = g_TabHistories[tab.inspectPtr];
                if (history.empty())
                {
                  if (tab.type == SearchTabState::TYPE_INSPECT_OBJECT)
                  {
                    Il2CppClass *klass = SafeGetClass(tab.inspectPtr);
                    const char *className = klass ? Il2cpp::GetClassName(klass) : "Unknown";
                    history.push_back({InspectHistoryNode::TYPE_OBJECT, tab.inspectPtr, klass, className});
                  }
                  else
                  {
                    history.push_back({InspectHistoryNode::TYPE_RAW, tab.inspectPtr, tab.inspectKlass, tab.inspectLabel});
                  }
                }

                RenderInspectBreadcrumbs(history);

                ImGui::BeginChild("##UnifiedInspectorContent", ImVec2(0, 0), true, ImGuiWindowFlags_HorizontalScrollbar);
                ApplyDragScroll();
                auto &currentNode = history.back();
                if (currentNode.type == InspectHistoryNode::TYPE_OBJECT)
                {
                  RenderObjectInspectorContent(currentNode.ptr, currentNode.klass, tab.inspectPtr);
                }
                else
                {
                  RenderRawStructInspectorContent(currentNode.ptr, currentNode.klass, tab.inspectPtr);
                }
                ImGui::EndChild();
              }
              else
              {
                ImGui::TextDisabled("No object selected for inspection.");
              }
            }
            else
            {

              // 2. Image selection row
              float comboW = availW * 0.52f;
              float allCheckW = ImGui::GetFrameHeight() + ImGui::GetStyle().ItemInnerSpacing.x + ImGui::CalcTextSize("All").x;
              float imageTextW = availW - comboW - allCheckW - (itemSpacing * 2);

              ImGui::SetNextItemWidth(comboW);
              const char *currentImageName = (g_SelectedImageIdx >= 0 && g_SelectedImageIdx < (int)g_MetadataCache.size()) ? g_MetadataCache[g_SelectedImageIdx].name.c_str() : "Select Image";
              ImGui::PushStyleColor(ImGuiCol_FrameBg, ImVec4(0.16f, 0.16f, 0.20f, 1.00f));
              if (ImGui::BeginCombo("##ImageCombo", currentImageName))
              {
                for (size_t i = 0; i < g_MetadataCache.size(); ++i)
                {
                  bool isSelected = (g_SelectedImageIdx == (int)i);
                  if (ImGui::Selectable(g_MetadataCache[i].name.c_str(), isSelected))
                  {
                    g_SelectedImageIdx = (int)i;
                    g_SearchAllImages = false;
                    g_FilterNeedsUpdate = true;
                  }
                }
                ImGui::EndCombo();
              }
              ImGui::PopStyleColor();

              ImGui::SameLine();
              ImGui::SetCursorPosX(ImGui::GetCursorPosX() + (imageTextW - ImGui::CalcTextSize("Image").x) * 0.5f);
              ImGui::TextUnformatted("Image");

              ImGui::SameLine();
              if (ImGui::Checkbox("All", &g_SearchAllImages))
              {
                g_FilterNeedsUpdate = true;
              }

              g_PatternSearch[0] = '\0';

              // 3. Filter Search Input Row
              float optionsBtnW = ImGui::CalcTextSize("Options").x + ImGui::GetStyle().FramePadding.x * 2.0f;
              float filterLabelW = ImGui::CalcTextSize("Filter").x;
              float filterInputW = availW - filterLabelW - optionsBtnW - (itemSpacing * 3);

              ImGui::SetNextItemWidth(filterInputW);
              ImGui::PushStyleColor(ImGuiCol_FrameBg, ImVec4(0.16f, 0.16f, 0.20f, 1.00f));
              if (ImGui::InputText("##FilterInput", g_FilterSearch, sizeof(g_FilterSearch)))
              {
                g_FilterNeedsUpdate = true;
              }
              ImGui::PopStyleColor();

              ImGui::SameLine();
              ImGui::TextUnformatted("Filter");

              ImGui::SameLine();
              ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.25f, 0.43f, 1.00f));
              ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.28f, 0.35f, 0.60f, 1.00f));
              if (ImGui::Button("Options", ImVec2(optionsBtnW, 0)))
              {
                g_ShowFilterOptionsWindow = true;
                g_FitFilterOptionsWindowNextOpen = true;
                ImVec2 buttonMin = ImGui::GetItemRectMin();
                ImVec2 buttonMax = ImGui::GetItemRectMax();
                g_FilterOptionsWindowPos = ImVec2(buttonMin.x, buttonMax.y + ImGui::GetStyle().ItemSpacing.y);
              }
              ImGui::PopStyleColor(2);

              // Filter Options floats like the object picker so it can receive touches outside the main menu rect.
              if (g_ShowFilterOptionsWindow)
              {
                if (g_FitFilterOptionsWindowNextOpen)
                {
                  ImGui::SetNextWindowPos(g_FilterOptionsWindowPos, ImGuiCond_Always);
                  ImGui::SetNextWindowSize(ImVec2(330.0f * g_FontScale, 0.0f), ImGuiCond_Always);
                  g_FitFilterOptionsWindowNextOpen = false;
                }
                ImGui::SetNextWindowSizeConstraints(
                    ImVec2(260.0f * g_FontScale, 160.0f * g_FontScale),
                    ImVec2(450.0f * g_FontScale, 520.0f * g_FontScale));
                ImGui::PushStyleColor(ImGuiCol_TitleBg, ImVec4(0.20f, 0.25f, 0.43f, 1.00f));
                ImGui::PushStyleColor(ImGuiCol_TitleBgActive, ImVec4(0.29f, 0.37f, 0.65f, 1.00f));
                if (ImGui::Begin("Filter Options##float", &g_ShowFilterOptionsWindow, ImGuiWindowFlags_NoCollapse | ImGuiWindowFlags_AlwaysAutoResize))
                {
                  ImVec2 fwPos = ImGui::GetWindowPos();
                  ImVec2 fwSize = ImGui::GetWindowSize();
                  s_filterRect = {true, fwPos.x, fwPos.y, fwSize.x, fwSize.y};

                  ImGui::TextUnformatted("Filter Options");
                  ImGui::Separator();
                  ImGui::Spacing();

                  if (ImGui::Checkbox("Case Sensitive", &g_FilterCaseSensitive))
                    g_FilterNeedsUpdate = true;
                  if (ImGui::Checkbox("Simple Mode", &g_FilterSimpleMode))
                    g_FilterNeedsUpdate = true;

                  ImGui::Spacing();
                  ImGui::TextUnformatted("Filter by:");
                  ImGui::SameLine();

                  ImGui::PushStyleColor(ImGuiCol_CheckMark, ImVec4(0.45f, 0.55f, 0.85f, 1.0f));
                  if (ImGui::RadioButton("Class", g_FilterType == 0))
                  {
                    g_FilterType = 0;
                    g_FilterNeedsUpdate = true;
                  }
                  ImGui::SameLine();
                  if (ImGui::RadioButton("Method", g_FilterType == 1))
                  {
                    g_FilterType = 1;
                    g_FilterNeedsUpdate = true;
                  }
                  ImGui::SameLine();
                  if (ImGui::RadioButton("Field", g_FilterType == 2))
                  {
                    g_FilterType = 2;
                    g_FilterNeedsUpdate = true;
                  }
                  ImGui::PopStyleColor();

                  if (g_FilterType == 1)
                  {
                    ImGui::Spacing();
                    ImGui::Separator();
                    ImGui::TextDisabled("Filter Method Options");
                    ImGui::PushStyleColor(ImGuiCol_CheckMark, ImVec4(0.45f, 0.55f, 0.85f, 1.0f));
                    if (ImGui::RadioButton("Return Type", g_FilterMethodOption == 0))
                    {
                      g_FilterMethodOption = 0;
                      g_FilterNeedsUpdate = true;
                    }
                    if (ImGui::RadioButton("Method Name", g_FilterMethodOption == 1))
                    {
                      g_FilterMethodOption = 1;
                      g_FilterNeedsUpdate = true;
                    }
                    if (ImGui::RadioButton("Method Parameter", g_FilterMethodOption == 2))
                    {
                      g_FilterMethodOption = 2;
                      g_FilterNeedsUpdate = true;
                    }
                    ImGui::PopStyleColor();
                  }

                  ImGui::Spacing();
                  if (ImGui::Checkbox("Must Starts With", &g_FilterStartsWith))
                    g_FilterNeedsUpdate = true;
                  if (ImGui::Checkbox("Must Ends With", &g_FilterEndsWith))
                    g_FilterNeedsUpdate = true;
                }
                else
                {
                  s_filterRect.active = false;
                }
                ImGui::End();
                ImGui::PopStyleColor(2);
              }
              else
              {
                s_filterRect.active = false;
              }

              // 5. Layout row
              float fullBtnW = 32.0f * g_FontScale;
              float splitCheckW = availW - fullBtnW - itemSpacing;

              ImGui::SetNextItemWidth(splitCheckW);
              if (ImGui::Checkbox("Split Layout", &g_SplitLayout))
              {
                SaveActiveSearchTabState();
              }
              ImGui::SameLine();
              ImGui::SetCursorPosX(availW - fullBtnW);
              RenderInspectorFullViewToggleButton("tools_search_full", ImVec2(fullBtnW, 0));

              ImGui::Spacing();
              ImGui::Separator();
              ImGui::Spacing();

              // 6. Apply filter if dirty
              if (g_FilterNeedsUpdate || strcmp(g_PatternSearch, g_LastPatternSearch) != 0 || strcmp(g_FilterSearch, g_LastFilterSearch) != 0)
              {
                UpdateFilteredMetadataUI();
                strncpy(g_LastPatternSearch, g_PatternSearch, sizeof(g_LastPatternSearch));
                strncpy(g_LastFilterSearch, g_FilterSearch, sizeof(g_LastFilterSearch));
                g_FilterNeedsUpdate = false;

                if (g_SelectedFlatIdx >= (int)g_FilteredList.size())
                {
                  g_SelectedFlatIdx = g_FilteredList.empty() ? -1 : 0;
                }
                RestoreSavedClassSelection();
                SaveActiveSearchTabState();
              }
              RestoreSavedClassSelection();

              // 7. Results list rendering
              if (g_SplitLayout)
              {
                float sidebarW = availW * 0.38f;
                float inspectorW = availW - sidebarW - itemSpacing;

                // Left Panel: Assembly/Class List
                ImGui::BeginChild("##LeftPanelTree", ImVec2(sidebarW, 0), true, ImGuiWindowFlags_HorizontalScrollbar);
                ApplyDragScroll();
                for (size_t i = 0; i < g_FilteredList.size(); ++i)
                {
                  auto &item = g_FilteredList[i];
                  bool isSelected = (g_SelectedFlatIdx == (int)i);

                  char itemLabel[512];
                  if (item.ns.empty())
                  {
                    snprintf(itemLabel, sizeof(itemLabel), "%s##cls_%zu", item.name.c_str(), i);
                  }
                  else
                  {
                    snprintf(itemLabel, sizeof(itemLabel), "%s.%s##cls_%zu", item.ns.c_str(), item.name.c_str(), i);
                  }

                  if (ImGui::Selectable(itemLabel, isSelected))
                  {
                    SetSelectedClassState((int)i, true);
                    SaveActiveSearchTabState();
                  }
                  ImGui::Separator();
                }
                ImGui::EndChild();

                ImGui::SameLine();

                // Right Panel: Selected Class Inspector (reuses the collapsible lists code)
                ImGui::BeginChild("##RightPanelInspector", ImVec2(inspectorW, 0), true, ImGuiWindowFlags_HorizontalScrollbar);
                ApplyDragScroll();
                if (g_SelectedFlatIdx >= 0 && g_SelectedFlatIdx < (int)g_FilteredList.size())
                {
                  auto &selectedItem = g_FilteredList[g_SelectedFlatIdx];

                  if (!selectedItem.ns.empty())
                  {
                    ImGui::TextColored(ImVec4(0.7f, 0.7f, 0.7f, 1.0f), "Namespace: %s", selectedItem.ns.c_str());
                  }

                  std::string classHeader = "public ";
                  if (selectedItem.isEnum)
                    classHeader += "enum ";
                  else if (selectedItem.isStruct)
                    classHeader += "struct ";
                  else
                    classHeader += "class ";
                  classHeader += selectedItem.name;
                  if (!selectedItem.parentName.empty())
                  {
                    classHeader += " : " + selectedItem.parentName;
                  }

                  ImGui::TextColored(ImVec4(0.4f, 0.8f, 1.0f, 1.0f), "%s", classHeader.c_str());
                  ImGui::Separator();
                  ImGui::Spacing();

                  RenderClassInspectorContent(selectedItem);
                }
                else
                {
                  ImGui::TextDisabled("Select an item from the left panel to inspect.");
                }
                ImGui::EndChild();
              }
              else
              {
                // Flat layout keeps the search results as classes, with the inspector below.
                bool hasSelectedClass = g_SelectedFlatIdx >= 0 && g_SelectedFlatIdx < (int)g_FilteredList.size();
                float listHeight = hasSelectedClass
                                       ? std::min(std::max(92.0f, ImGui::GetContentRegionAvail().y * 0.26f), 170.0f)
                                       : 0.0f;
                ImGui::BeginChild("##FlatListPanel", ImVec2(0, listHeight), true, ImGuiWindowFlags_HorizontalScrollbar);
                ApplyDragScroll();
                for (size_t i = 0; i < g_FilteredList.size(); ++i)
                {
                  auto &item = g_FilteredList[i];
                  bool isSelected = (g_SelectedFlatIdx == (int)i);

                  char itemLabel[1024];
                  if (item.ns.empty())
                  {
                    snprintf(itemLabel, sizeof(itemLabel), "%s##item_%zu", item.name.c_str(), i);
                  }
                  else
                  {
                    snprintf(itemLabel, sizeof(itemLabel), "%s.%s##item_%zu", item.ns.c_str(), item.name.c_str(), i);
                  }

                  if (ImGui::Selectable(itemLabel, isSelected))
                  {
                    SetSelectedClassState((int)i, true);
                    SaveActiveSearchTabState();
                  }
                  ImGui::Separator();
                }
                ImGui::EndChild();

                if (hasSelectedClass)
                {
                  auto &selectedItem = g_FilteredList[g_SelectedFlatIdx];
                  ImGui::Spacing();

                  float btnW = ImGui::GetFrameHeight();
                  float traceAllW = ImGui::CalcTextSize("Trace All").x + ImGui::GetStyle().FramePadding.x * 2.0f;
                  float restoreAllW = ImGui::CalcTextSize("Restore All").x + ImGui::GetStyle().FramePadding.x * 2.0f;
                  float traceBtnW = std::max(traceAllW, restoreAllW);
                  float rightControlsW = traceBtnW + btnW + itemSpacing;

                  // Filtered checkbox on the left
                  ImGui::Checkbox("Filtered", &g_InspectorFiltered);

                  ImGui::SameLine();
                  float rightX = ImGui::GetWindowWidth() - ImGui::GetStyle().WindowPadding.x - rightControlsW;
                  ImGui::SetCursorPosX(std::max(ImGui::GetCursorPosX(), rightX));

                  ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.25f, 0.43f, 1.00f));
                  ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.28f, 0.35f, 0.60f, 1.00f));
                  bool classTraced = IsAnyMethodTraced(selectedItem);
                  if (ImGui::Button(classTraced ? "Restore All" : "Trace All", ImVec2(traceBtnW, 0)))
                  {
                    if (classTraced)
                      RestoreClassMethods(selectedItem);
                    else
                      TraceClassMethods(selectedItem);
                  }
                  ImGui::PopStyleColor(2);

                  ImGui::SameLine();
                  ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.20f, 0.25f, 0.43f, 1.0f));
                  ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.28f, 0.35f, 0.60f, 1.0f));
                  if (ImGui::Button("X", ImVec2(btnW, 0)))
                  {
                    g_SelectedFlatIdx = -1;
                    SaveConfig();
                    SaveActiveSearchTabState();
                  }
                  ImGui::PopStyleColor(2);

                  // 2. Separator button banner "Inspector" (matches reference tool)
                  ImGui::Spacing();
                  ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(0.17f, 0.22f, 0.36f, 1.0f));
                  ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(0.24f, 0.31f, 0.51f, 1.0f));
                  ImGui::PushStyleColor(ImGuiCol_ButtonActive, ImVec4(0.30f, 0.39f, 0.64f, 1.0f));
                  if (ImGui::Button("Inspector", ImVec2(-FLT_MIN, 0)))
                  {
                    g_ShowInstancesWindow = true;
                    g_ScanClassName = BuildClassFullName(selectedItem);
                    g_FitInstancesWindowNextOpen = true;
                  }
                  ImGui::PopStyleColor(3);

                  ImGui::BeginChild("##FlatInspectorContent", ImVec2(0, 0), false, ImGuiWindowFlags_HorizontalScrollbar);
                  ApplyDragScroll();
                  RenderClassInspectorContent(selectedItem);
                  ImGui::EndChild();
                }
              }
            }
          }
          ImGui::EndTabItem();
        }

        if (!g_SilentMode && g_ShowTabTracer && ImGui::BeginTabItem("Tracer"))
        {
          g_ActiveTabIndex = 6;
          RenderTraceTabContent();
          ImGui::EndTabItem();
        }

        if (ImGui::BeginTabItem("Settings"))
        {
          g_ActiveTabIndex = 2;
          LoadConfig();

          if (ImGui::CollapsingHeader("General", ImGuiTreeNodeFlags_DefaultOpen))
          {
            ImGui::Spacing();
            ImGui::TextDisabled("Display");
            ImGui::Spacing();

            // 1. Scale option row
            float scales[] = {0.50f, 0.75f, 1.00f, 1.25f, 1.50f, 1.75f, 2.00f, 2.25f, 2.50f};
            const char *labels[] = {"50%", "75%", "100%", "125%", "150%", "175%", "200%", "225%", "250%"};
            const char *currentScaleLabel = "100%";
            for (int i = 0; i < 9; ++i)
            {
              if (g_FontScale == scales[i])
              {
                currentScaleLabel = labels[i];
                break;
              }
            }

            ImGui::PushItemWidth(180.0f);
            ImGui::InputText("##scale_preview", (char *)currentScaleLabel, strlen(currentScaleLabel), ImGuiInputTextFlags_ReadOnly);
            ImGui::PopItemWidth();
            ImGui::SameLine();

            ImGui::PushStyleColor(ImGuiCol_Button, ImGui::GetStyle().Colors[ImGuiCol_HeaderActive]);
            if (ImGui::ArrowButton("##scale_dropdown", ImGuiDir_Down))
            {
              ImGui::OpenPopup("##ScalePopup");
            }
            ImGui::PopStyleColor();
            ImGui::SameLine();
            ImGui::TextUnformatted("Scale");

            if (ImGui::BeginPopup("##ScalePopup"))
            {
              for (int i = 0; i < 9; ++i)
              {
                if (ImGui::Selectable(labels[i], g_FontScale == scales[i]))
                {
                  ApplyMenuFontScale(scales[i]);
                  SaveConfig();
                }
              }
              ImGui::EndPopup();
            }

            if (!g_SilentMode)
            {
              // 2. Style option row (Placeholder)
              ImGui::PushItemWidth(180.0f);
              ImGui::InputText("##style_preview", (char *)"Default", 7, ImGuiInputTextFlags_ReadOnly);
              ImGui::PopItemWidth();
              ImGui::SameLine();
              ImGui::PushStyleColor(ImGuiCol_Button, ImGui::GetStyle().Colors[ImGuiCol_HeaderActive]);
              if (ImGui::ArrowButton("##style_dropdown", ImGuiDir_Down))
              {
                // Placeholder
              }
              ImGui::PopStyleColor();
              ImGui::SameLine();
              ImGui::TextUnformatted("Style");

              // 3. Language option row (Placeholder)
              ImGui::PushItemWidth(180.0f);
              ImGui::InputText("##lang_preview", (char *)"English", 7, ImGuiInputTextFlags_ReadOnly);
              ImGui::PopItemWidth();
              ImGui::SameLine();
              ImGui::PushStyleColor(ImGuiCol_Button, ImGui::GetStyle().Colors[ImGuiCol_HeaderActive]);
              if (ImGui::ArrowButton("##lang_dropdown", ImGuiDir_Down))
              {
                // Placeholder
              }
              ImGui::PopStyleColor();
              ImGui::SameLine();
              ImGui::TextUnformatted("Language");
            }

            // 4. Checkbox / controls row
            if (ImGui::Checkbox("Show Menu Toggler", &g_ShowMenuToggler))
            {
              SaveConfig();
            }

            if (!g_SilentMode)
            {
              ImGui::Spacing();
              ImGui::TextDisabled("Tabs");
              ImGui::Spacing();

              // Tabs box child window
              ImGui::PushStyleVar(ImGuiStyleVar_ChildRounding, 4.0f);
              float boxHeight = ImGui::GetFrameHeightWithSpacing() * 2.0f + ImGui::GetStyle().WindowPadding.y * 2.0f;
              if (ImGui::BeginChild("##TabsBox", ImVec2(0.0f, boxHeight), true, ImGuiWindowFlags_NoScrollbar))
              {
                // Column 1
                ImGui::BeginGroup();
                if (ImGui::Checkbox("Scripting", &g_ShowTabScript))
                {
                  SaveConfig();
                }
                if (ImGui::Checkbox("Patches", &g_ShowTabPatched))
                {
                  SaveConfig();
                }
                ImGui::EndGroup();

                ImGui::SameLine(ImGui::GetContentRegionAvail().x * 0.5f);

                // Column 2
                ImGui::BeginGroup();
                if (ImGui::Checkbox("Trace", &g_ShowTabTracer))
                {
                  SaveConfig();
                }
                if (ImGui::Checkbox("Dumper", &g_ShowTabDump))
                {
                  SaveConfig();
                }
                ImGui::EndGroup();
              }
              ImGui::EndChild();
              ImGui::PopStyleVar();
            }

            // Unified Access Key Switch Section at the bottom
            ImGui::Spacing();
            ImGui::Separator();
            ImGui::Spacing();
            ImGui::TextDisabled("Account");
            ImGui::Spacing();

            static char s_modeKeyInput[64] = "";
            ImGui::PushItemWidth(180.0f);
            bool hitEnter = ImGui::InputText("##ModeKeyInput", s_modeKeyInput, sizeof(s_modeKeyInput), ImGuiInputTextFlags_Password | ImGuiInputTextFlags_EnterReturnsTrue);
            ImGui::PopItemWidth();
            ImGui::SameLine();

            if (ImGui::Button("Switch account") || hitEnter)
            {
              if (strcmp(s_modeKeyInput, "V7") == 0)
              {
                g_SilentMode = false;
                strncpy(g_KeyInput, "V7", sizeof(g_KeyInput));
                strncpy(g_SavedKey, "V7", sizeof(g_SavedKey));
                SaveConfig();
                s_modeKeyInput[0] = '\0';
              }
              else if (strcmp(s_modeKeyInput, "cilent") == 0 || strcmp(s_modeKeyInput, "client") == 0)
              {
                g_SilentMode = true;
                strncpy(g_KeyInput, s_modeKeyInput, sizeof(g_KeyInput));
                strncpy(g_SavedKey, s_modeKeyInput, sizeof(g_SavedKey));
                SaveConfig();
                s_modeKeyInput[0] = '\0';
              }
            }
          }

          ImGui::EndTabItem();
        }
      }
      ImGui::EndTabBar();
    }

    // Social footer removed for the GranSaga Idle build.
  }

  if (g_CallerObjectPickerOpen)
  {
    if (g_FitCallerObjectPickerNextOpen)
    {
      ImGui::SetNextWindowSize(ImVec2(360 * g_FontScale, 330 * g_FontScale), ImGuiCond_Always);
      g_FitCallerObjectPickerNextOpen = false;
    }
    else
    {
      ImGui::SetNextWindowSize(ImVec2(360 * g_FontScale, 330 * g_FontScale), ImGuiCond_FirstUseEver);
    }
    ImGui::SetNextWindowSizeConstraints(ImVec2(300 * g_FontScale, 220 * g_FontScale), ImVec2(520 * g_FontScale, 520 * g_FontScale));
    ImGui::SetNextWindowPos(ImVec2(180 * g_FontScale, 130 * g_FontScale), ImGuiCond_FirstUseEver);
    if (ImGui::Begin("Caller Object Picker", &g_CallerObjectPickerOpen))
    {
      ImVec2 fwPos = ImGui::GetWindowPos();
      ImVec2 fwSize = ImGui::GetWindowSize();
      s_callerPickerRect = {true, fwPos.x, fwPos.y, fwSize.x, fwSize.y};

      ImGui::Text("Find %s", g_CallerObjectPickerTitle.c_str());
      ImGui::Separator();
      if (ImGui::Button("Find Objects", ImVec2(150.0f, 0)))
      {
        g_ScanClassName = g_CallerObjectPickerClassName;
        g_ScannedObjects.clear();
        g_ScanRequested = true;
      }
      ImGui::SameLine();
      static char callerManualAddress[32] = "";
      ImGui::SetNextItemWidth(180.0f);
      ImGui::InputText("##callerManualAddr", callerManualAddress, sizeof(callerManualAddress));
      ImGui::SameLine();
      if (ImGui::Button("Manual Input"))
      {
        unsigned long long addr = 0;
        if (sscanf(callerManualAddress, "%llX", &addr) == 1 ||
            sscanf(callerManualAddress, "0x%llX", &addr) == 1)
        {
          SetCallerObjectSelection(g_CallerObjectPickerSlotKey, (uintptr_t)addr);
          g_CallerObjectPickerOpen = false;
        }
      }

      if (g_ScanRequested || g_ScanInProgress)
      {
        ImGui::TextColored(ImVec4(1.0f, 0.8f, 0.2f, 1.0f), "Scanning...");
      }

      ImGui::Text("%s (%zu)", g_CallerObjectPickerClassName.c_str(), g_ScannedObjects.size());
      ImGui::BeginChild("##CallerObjectList", ImVec2(0, 0), true, ImGuiWindowFlags_HorizontalScrollbar);

      if (ImGui::Selectable("nil (null)", false))
      {
        SetCallerObjectSelection(g_CallerObjectPickerSlotKey, 0);
        g_CallerObjectPickerOpen = false;
      }
      ImGui::Separator();

      if (g_ScannedObjects.empty())
      {
        ImGui::TextDisabled("No objects. Click Find Objects.");
      }
      else
      {
        for (size_t objIndex = 0; objIndex < g_ScannedObjects.size(); ++objIndex)
        {
          void *obj = g_ScannedObjects[objIndex];
          if (!obj)
            continue;
          char label[192];
          snprintf(label, sizeof(label), "%s 0x%llX",
                   g_CallerObjectPickerTitle.c_str(), (unsigned long long)obj);
          // Always visually highlight the first item (index 0) as default hint
          bool isDefault = (objIndex == 0);
          if (ImGui::Selectable(label, isDefault))
          {
            SetCallerObjectSelection(g_CallerObjectPickerSlotKey, (uintptr_t)obj);
            g_CallerObjectPickerOpen = false;
          }
        }
      }
      ImGui::EndChild();

      ImGui::Separator();
      Il2CppClass *pickKlass = Il2cpp::FindClass(g_CallerObjectPickerClassName.c_str());
      if (pickKlass)
      {
        if (ImGui::CollapsingHeader("Create New Instance", ImGuiTreeNodeFlags_None))
        {
          void *iter = nullptr;
          int ctorIndex = 0;
          while (auto m = Il2cpp::GetClassMethods(pickKlass, &iter))
          {
            if (strcmp(Il2cpp::GetMethodName(m), ".ctor") != 0)
              continue;

            ctorIndex++;
            ImGui::PushID(ctorIndex);

            std::string ctorLabel = "new (";
            int paramCount = Il2cpp::GetMethodParamCount(m);
            for (int i = 0; i < paramCount; ++i)
            {
              Il2CppType *pType = Il2cpp::GetMethodParam(m, i);
              const char *pTypeName = Il2cpp::GetTypeName(pType);
              if (i > 0)
                ctorLabel += ", ";
              ctorLabel += pTypeName ? pTypeName : "object";
            }
            ctorLabel += ")";

            if (ImGui::TreeNode(ctorLabel.c_str()))
            {
              // Inputs for constructor parameters
              static char ctorArgs[16][128] = {};
              static int ctorInts[16] = {};
              static float ctorFloats[16] = {};
              static double ctorDoubles[16] = {};
              static bool ctorBools[16] = {};

              for (int p = 0; p < paramCount; ++p)
              {
                Il2CppType *param = Il2cpp::GetMethodParam(m, p);
                std::string label = std::string(Il2cpp::GetTypeName(param)) + " arg" + std::to_string(p);
                ImGui::PushID(p);

                if (IsBoolTypeName(Il2cpp::GetTypeName(param)))
                {
                  int choice = ctorBools[p] ? 1 : 0;
                  const char *items[] = {"false", "true"};
                  ImGui::SetNextItemWidth(120.0f * g_FontScale);
                  if (ImGui::Combo(label.c_str(), &choice, items, 2))
                    ctorBools[p] = choice != 0;
                }
                else if (IsIntTypeName(Il2cpp::GetTypeName(param)) || IsLongTypeName(Il2cpp::GetTypeName(param)))
                {
                  ImGui::SetNextItemWidth(120.0f * g_FontScale);
                  ImGui::InputInt(label.c_str(), &ctorInts[p]);
                }
                else if (IsFloatTypeName(Il2cpp::GetTypeName(param)))
                {
                  ImGui::SetNextItemWidth(120.0f * g_FontScale);
                  ImGui::InputFloat(label.c_str(), &ctorFloats[p]);
                }
                else if (IsDoubleTypeName(Il2cpp::GetTypeName(param)))
                {
                  ImGui::SetNextItemWidth(120.0f * g_FontScale);
                  ImGui::InputDouble(label.c_str(), &ctorDoubles[p]);
                }
                else if (IsStringTypeName(Il2cpp::GetTypeName(param)))
                {
                  ImGui::SetNextItemWidth(180.0f * g_FontScale);
                  ImGui::InputText(label.c_str(), ctorArgs[p], sizeof(ctorArgs[p]));
                }
                else
                {
                  // Object parameter selection
                  std::string paramSlotKey = "ctor_pick_" + std::to_string(ctorIndex) + "_arg_" + std::to_string(p);
                  uintptr_t selectedParamObj = g_CallerObjectSelections[paramSlotKey];
                  std::string argComboPreview = selectedParamObj ? FormatObjectAddress(selectedParamObj) : "nil";
                  ImGui::SetNextItemWidth(180.0f * g_FontScale);
                  if (ImGui::BeginCombo(label.c_str(), argComboPreview.c_str()))
                  {
                    OpenCallerObjectPicker(Il2cpp::GetTypeName(param), label, paramSlotKey);
                    ImGui::EndCombo();
                  }
                }
                ImGui::PopID();
              }

              if (ImGui::Button("Create Instance"))
              {
                // Build Lua Script to call: Class.fromName("ClassName"):new(...)
                std::string callScript = "local klass = Class.fromName(\"" + g_CallerObjectPickerClassName + "\")\n";
                callScript += "local obj = klass:new(";
                for (int p = 0; p < paramCount; ++p)
                {
                  if (p > 0)
                    callScript += ", ";
                  Il2CppType *param = Il2cpp::GetMethodParam(m, p);
                  if (IsBoolTypeName(Il2cpp::GetTypeName(param)))
                  {
                    callScript += ctorBools[p] ? "true" : "false";
                  }
                  else if (IsIntTypeName(Il2cpp::GetTypeName(param)) || IsLongTypeName(Il2cpp::GetTypeName(param)))
                  {
                    callScript += std::to_string(ctorInts[p]);
                  }
                  else if (IsFloatTypeName(Il2cpp::GetTypeName(param)))
                  {
                    callScript += std::to_string(ctorFloats[p]);
                  }
                  else if (IsDoubleTypeName(Il2cpp::GetTypeName(param)))
                  {
                    callScript += std::to_string(ctorDoubles[p]);
                  }
                  else if (IsStringTypeName(Il2cpp::GetTypeName(param)))
                  {
                    callScript += LuaQuote(ctorArgs[p]);
                  }
                  else
                  {
                    std::string paramSlotKey = "ctor_pick_" + std::to_string(ctorIndex) + "_arg_" + std::to_string(p);
                    uintptr_t selectedParamObj = g_CallerObjectSelections[paramSlotKey];
                    callScript += selectedParamObj ? ObjectExprFromAddress(selectedParamObj) : "nil";
                  }
                }
                callScript += ")\n";
                callScript += "if obj then\n";
                callScript += "  registerCreatedObject(obj, \"" + g_CallerObjectPickerSlotKey + "\")\n";
                callScript += "  print(\"Created instance and selected for parameter\")\n";
                callScript += "else\n";
                callScript += "  print(\"Failed to create instance\")\n";
                callScript += "end\n";

                LuaConsole_RequestExecute(callScript.c_str());
                g_CallerObjectPickerOpen = false;
              }

              ImGui::TreePop();
            }

            ImGui::PopID();
          }
        }
      }
    }
    else
    {
      s_callerPickerRect.active = false;
    }
    ImGui::End();
  }
  else
  {
    s_callerPickerRect.active = false;
  }

  RenderInstancesWindow();
  RenderRedirectChooserWindow();

  // Centralized float window bounds update
  float fx = 0.0f, fy = 0.0f, fw = 0.0f, fh = 0.0f;
  auto unionRect = [&](float x2, float y2, float w2, float h2)
  {
    if (w2 <= 0.0f || h2 <= 0.0f)
      return;
    if (fw <= 0.0f || fh <= 0.0f)
    {
      fx = x2;
      fy = y2;
      fw = w2;
      fh = h2;
      return;
    }
    float minX = std::min(fx, x2);
    float minY = std::min(fy, y2);
    float maxX = std::max(fx + fw, x2 + w2);
    float maxY = std::max(fy + fh, y2 + h2);
    fx = minX;
    fy = minY;
    fw = maxX - minX;
    fh = maxY - minY;
  };

  if (!g_ShowInstancesWindow && !g_CallerObjectPickerOpen && !g_ScannedObjects.empty())
  {
    g_ScannedObjects.clear();
  }

  if (s_redirectRect.active)
    unionRect(s_redirectRect.x, s_redirectRect.y, s_redirectRect.w, s_redirectRect.h);
  if (s_instancesRect.active)
    unionRect(s_instancesRect.x, s_instancesRect.y, s_instancesRect.w, s_instancesRect.h);
  if (s_filterRect.active)
    unionRect(s_filterRect.x, s_filterRect.y, s_filterRect.w, s_filterRect.h);
  if (s_callerPickerRect.active)
    unionRect(s_callerPickerRect.x, s_callerPickerRect.y, s_callerPickerRect.w, s_callerPickerRect.h);

  UpdateOverlayFloatWinBounds(fx, fy, fw, fh);

  ImGui::End();
  ImGui::PopStyleColor(20);
  ImGui::PopStyleVar(7);

  if (ImGui::GetIO().WantSaveIniSettings)
  {
    ImGui::GetIO().WantSaveIniSettings = false;
    SaveConfig();
  }
}

void UpdateImGuiTouchInput(ImGuiIO &io)
{
  static bool s_lastWantTextInput = false;
  bool wantTextInput = ImGui::GetIO().WantTextInput;
  if (wantTextInput != s_lastWantTextInput)
  {
    s_lastWantTextInput = wantTextInput;
    ShowKeyboard(wantTextInput);
  }
  static MethodInfo *TouchCountMethod = nullptr;
  static MethodInfo *GetTouchMethodInfo = nullptr;

  if (!TouchCountMethod)
  {
    TouchCountMethod = Il2cpp::FindMethod(OBFUSCATE("UnityEngine.Input"), OBFUSCATE("get_touchCount"), 0);
  }

  if (!GetTouchMethodInfo)
  {
    GetTouchMethodInfo = Il2cpp::FindMethod(OBFUSCATE("UnityEngine.Input"), OBFUSCATE("GetTouch"), 1);
  }

  if (TouchCountMethod && GetTouchMethodInfo)
  {
    int touchCount = TouchCountMethod->invoke_static<int>();
    if (touchCount > 0)
    {
      auto touch = GetTouchMethodInfo->invoke_static<UnityEngine_Touch_Fields>(0);
      float reverseY = io.DisplaySize.y - touch.m_Position.fields.y;

      switch (touch.m_Phase)
      {
      case TouchPhase::Began:
      case TouchPhase::Stationary:
        io.MousePos = ImVec2(touch.m_Position.fields.x, reverseY);
        io.MouseDown[0] = true;
        break;
      case TouchPhase::Ended:
      case TouchPhase::Canceled:
        io.MouseDown[0] = false;
        should_clear_mouse_pos = true;
        break;
      case TouchPhase::Moved:
        io.MousePos = ImVec2(touch.m_Position.fields.x, reverseY);
        break;
      default:
        break;
      }
    }
    else
    {
      io.MouseDown[0] = false;
    }
  }
}

EGLBoolean (*orig_eglSwapBuffers)(EGLDisplay dpy, EGLSurface surface);
static volatile bool g_EglFallbackHookInstalled = false;
static volatile bool g_GameRenderFallbackActive = false;

bool IsGameRenderFallbackActive()
{
  return g_GameRenderFallbackActive;
}

void SetGameRenderFallbackActive(bool active)
{
  g_GameRenderFallbackActive = active;
}

EGLBoolean _eglSwapBuffers(EGLDisplay dpy, EGLSurface surface)
{
  if (IsOverlaySurfaceReady())
  {
    if (IsGameRenderFallbackActive())
      SetGameRenderFallbackActive(false);
    return orig_eglSwapBuffers ? orig_eglSwapBuffers(dpy, surface) : eglSwapBuffers(dpy, surface);
  }

  eglQuerySurface(dpy, surface, EGL_WIDTH, &glWidth);
  eglQuerySurface(dpy, surface, EGL_HEIGHT, &glHeight);
  if (glWidth <= 0 || glHeight <= 0)
  {
    return orig_eglSwapBuffers ? orig_eglSwapBuffers(dpy, surface) : eglSwapBuffers(dpy, surface);
  }

  if (!setup)
  {
    LOGI("EGLFallback: initializing ImGui on eglSwapBuffers %dx%d", glWidth, glHeight);
    IMGUI_CHECKVERSION();
    ImGui::CreateContext();
    ImGuiIO &io = ImGui::GetIO();
    ImGui::StyleColorsDark();
    ImGuiStyle &style = ImGui::GetStyle();
    style.WindowTitleAlign = ImVec2(0.5f, 0.5f);
    style.FrameRounding = 5.0f;
    style.GrabRounding = 4.0f;
    style.WindowRounding = 6.0f;
    style.ChildRounding = 5.0f;
    style.ScrollbarSize = 22.0f;
    style.ScrollbarRounding = 11.0f;
    style.FrameBorderSize = 1.0f;
    style.WindowBorderSize = 1.0f;
    style.PopupBorderSize = 1.0f;
    style.PopupRounding = 5.0f;

    // Màu nền window
    style.Colors[ImGuiCol_WindowBg] = ImVec4(0.06f, 0.06f, 0.09f, 0.96f);
    // Màu title bar
    style.Colors[ImGuiCol_TitleBg] = ImVec4(0.20f, 0.25f, 0.43f, 1.00f);
    style.Colors[ImGuiCol_TitleBgActive] = ImVec4(0.29f, 0.37f, 0.65f, 1.00f);
    // Màu frame (background của input, checkbox, slider)
    style.Colors[ImGuiCol_FrameBg] = ImVec4(0.09f, 0.11f, 0.17f, 1.00f);
    style.Colors[ImGuiCol_FrameBgHovered] = ImVec4(0.14f, 0.17f, 0.26f, 1.00f);
    style.Colors[ImGuiCol_FrameBgActive] = ImVec4(0.20f, 0.25f, 0.38f, 1.00f);
    // Màu button
    style.Colors[ImGuiCol_Button] = ImVec4(0.20f, 0.25f, 0.43f, 1.00f);
    style.Colors[ImGuiCol_ButtonHovered] = ImVec4(0.28f, 0.35f, 0.60f, 1.00f);
    style.Colors[ImGuiCol_ButtonActive] = ImVec4(0.35f, 0.44f, 0.75f, 1.00f);
    // Màu checkmark
    style.Colors[ImGuiCol_CheckMark] = ImVec4(0.29f, 0.37f, 0.65f, 1.00f);
    // Màu slider grab
    style.Colors[ImGuiCol_SliderGrab] = ImVec4(0.29f, 0.37f, 0.65f, 1.00f);
    style.Colors[ImGuiCol_SliderGrabActive] = ImVec4(0.35f, 0.44f, 0.75f, 1.00f);
    // Màu header, separator
    style.Colors[ImGuiCol_Header] = ImVec4(0.20f, 0.25f, 0.43f, 1.00f);
    style.Colors[ImGuiCol_HeaderHovered] = ImVec4(0.28f, 0.35f, 0.60f, 1.00f);
    style.Colors[ImGuiCol_HeaderActive] = ImVec4(0.35f, 0.44f, 0.75f, 1.00f);
    style.Colors[ImGuiCol_Separator] = ImVec4(0.25f, 0.32f, 0.55f, 0.50f);
    style.Colors[ImGuiCol_SeparatorHovered] = ImVec4(0.29f, 0.37f, 0.65f, 0.70f);
    style.Colors[ImGuiCol_SeparatorActive] = ImVec4(0.35f, 0.44f, 0.75f, 1.00f);
    // Màu text
    style.Colors[ImGuiCol_Text] = ImVec4(0.94f, 0.94f, 0.96f, 1.00f);
    style.Colors[ImGuiCol_TextDisabled] = ImVec4(0.50f, 0.53f, 0.65f, 1.00f);
    // Màu border, resize grip
    style.Colors[ImGuiCol_Border] = ImVec4(0.25f, 0.32f, 0.55f, 0.80f);
    style.Colors[ImGuiCol_ResizeGrip] = ImVec4(0.20f, 0.25f, 0.43f, 0.40f);
    style.Colors[ImGuiCol_ResizeGripHovered] = ImVec4(0.28f, 0.35f, 0.60f, 0.60f);
    style.Colors[ImGuiCol_ResizeGripActive] = ImVec4(0.35f, 0.44f, 0.75f, 0.80f);
    // Scrollbar
    style.Colors[ImGuiCol_ScrollbarBg] = ImVec4(0.06f, 0.06f, 0.09f, 0.50f);
    style.Colors[ImGuiCol_ScrollbarGrab] = ImVec4(0.20f, 0.25f, 0.43f, 0.50f);
    style.Colors[ImGuiCol_ScrollbarGrabHovered] = ImVec4(0.28f, 0.35f, 0.60f, 0.70f);
    style.Colors[ImGuiCol_ScrollbarGrabActive] = ImVec4(0.35f, 0.44f, 0.75f, 0.90f);

    ImGui_ImplOpenGL3_Init("#version 300 es");
    SetupMenuFonts(io, 28.0f);
    ImGui::GetStyle().ScaleAllSizes(1.0f);
    CaptureBaseMenuStyle();
    LoadConfig();
    setup = true;
    SetGameRenderFallbackActive(true);
    LOGI("EGLFallback: render ready");
  }
  ImGuiIO &io = ImGui::GetIO();
  static bool s_lastWantTextInput = false;
  bool wantTextInput = ImGui::GetIO().WantTextInput;
  if (wantTextInput != s_lastWantTextInput)
  {
    s_lastWantTextInput = wantTextInput;
    ShowKeyboard(wantTextInput);
  }
  static MethodInfo *TouchCountMethod = nullptr;
  static MethodInfo *GetTouchMethodInfo = nullptr;

  if (!TouchCountMethod)
  {
    TouchCountMethod = Il2cpp::FindMethod(OBFUSCATE("UnityEngine.Input"), OBFUSCATE("get_touchCount"), 0);
  }

  if (!GetTouchMethodInfo)
  {
    GetTouchMethodInfo = Il2cpp::FindMethod(OBFUSCATE("UnityEngine.Input"), OBFUSCATE("GetTouch"), 1);
  }

  if (TouchCountMethod && GetTouchMethodInfo)
  {
    int touchCount = TouchCountMethod->invoke_static<int>();
    if (touchCount > 0)
    {
      auto touch = GetTouchMethodInfo->invoke_static<UnityEngine_Touch_Fields>(0);
      float reverseY = io.DisplaySize.y - touch.m_Position.fields.y;

      switch (touch.m_Phase)
      {
      case TouchPhase::Began:
      case TouchPhase::Stationary:
        io.MousePos = ImVec2(touch.m_Position.fields.x, reverseY);
        io.MouseDown[0] = true;
        break;
      case TouchPhase::Ended:
      case TouchPhase::Canceled:
        io.MouseDown[0] = false;
        should_clear_mouse_pos = true;
        break;
      case TouchPhase::Moved:
        io.MousePos = ImVec2(touch.m_Position.fields.x, reverseY);
        break;
      default:
        break;
      }
    }
    else
    {
      io.MouseDown[0] = false;
    }
  }
  // FBO fix: imgui_impl_opengl3 RenderDrawData đã tự save/restore
  // depth, stencil, blend, scissor, viewport, program, texture...
  // (xem imgui_impl_opengl3.cpp dòng 498-662).
  // Chỉ có FBO là KHÔNG được backend xử lý, nên ta phải tự làm.
  // KHÔNG glDisable(GL_DEPTH_TEST) trước RenderDrawData vì nó sẽ
  // save "disabled" → restore "disabled" → game mất depth → đứng.
  // ============================================================
  GLint last_framebuffer;
  glGetIntegerv(GL_FRAMEBUFFER_BINDING, &last_framebuffer);

  // Bind về default framebuffer — nếu game đang render vào FBO phụ,
  // ImGui sẽ vẽ vào đó thay vì lên màn hình → bị "nuốt"
  if (last_framebuffer != 0)
    glBindFramebuffer(GL_FRAMEBUFFER, 0);

  ImGui_ImplOpenGL3_NewFrame();
  io.DisplaySize = ImVec2((float)glWidth, (float)glHeight);
  ImGui::NewFrame();
  {
    std::lock_guard<std::mutex> lock(g_Il2cppMutex);
    BeginDraw();
  }
  ImGui::EndFrame();
  ImGui::Render();
  ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());

  // Khôi phục FBO nếu game đang dùng FBO khác
  if (last_framebuffer != 0)
    glBindFramebuffer(GL_FRAMEBUFFER, last_framebuffer);

  return orig_eglSwapBuffers(dpy, surface);
}

static void *EglFallbackHookThread(void *)
{
  void *lib = nullptr;
  for (int i = 0; i < 60 && !lib; ++i)
  {
    lib = dlopen("libEGL.so", RTLD_NOW | RTLD_NOLOAD);
    if (!lib)
      lib = dlopen("libEGL.so", RTLD_NOW);
    if (!lib)
    {
      sleep(1);
      continue;
    }
  }

  if (!lib)
  {
    LOGW("EGLFallback: libEGL.so not loaded, hook skipped");
    return nullptr;
  }

  void *swap = dlsym(lib, "eglSwapBuffers");
  if (!swap)
  {
    LOGW("EGLFallback: eglSwapBuffers not found");
    return nullptr;
  }

  if (DobbyHook(swap, (void *)_eglSwapBuffers, (void **)&orig_eglSwapBuffers) == RS_SUCCESS && orig_eglSwapBuffers)
  {
    g_EglFallbackHookInstalled = true;
    LOGI("EGLFallback: hooked eglSwapBuffers at %p old=%p", swap, orig_eglSwapBuffers);
  }
  else
  {
    LOGE("EGLFallback: DobbyHook eglSwapBuffers failed");
  }
  return nullptr;
}

static void StartEglFallbackHook()
{
  static bool started = false;
  if (started)
    return;
  started = true;
  pthread_t thread;
  pthread_create(&thread, nullptr, EglFallbackHookThread, nullptr);
  pthread_detach(thread);
}

// ============================================================
// Main Thread Hook: Time.deltaTime chạy trên Unity main thread
// ============================================================
// Keep this as a safe place to run future main-thread feature logic.
static void MainLivenessRootCallback(void *data, void *userData)
{
  void *state = userData;
  if (data && state && il2cpp_unity_liveness_calculation_from_root)
  {
    il2cpp_unity_liveness_calculation_from_root((Il2CppObject *)data, state);
  }
}

float (*orig_Time_get_deltaTime)(MethodInfo *method);

float my_Time_get_deltaTime(MethodInfo *method)
{
  // Gọi original trước để không ảnh hưởng game
  float result = orig_Time_get_deltaTime(method);

  clock_t now = clock();
  double tickLogElapsed = (double)(now - mainThread_LastTickLogTime) / CLOCKS_PER_SEC;
  if (tickLogElapsed >= 5.0 || mainThread_LastTickLogTime == 0)
  {
    LOGD("MainThread: Time.get_deltaTime tick");
    mainThread_LastTickLogTime = now;
  }

  if (g_ScanRequested && !g_ScanInProgress)
  {
    std::lock_guard<std::mutex> lock(g_Il2cppMutex);
    g_ScanInProgress = true;
    ClearScannedObjects();
    std::string resolvedScanName;
    Il2CppClass *targetKlass = FindClassForObjectScan(g_ScanClassName, &resolvedScanName);
    if (targetKlass)
    {
      LOGI("ObjectScan: requested=%s resolved=%s", g_ScanClassName.c_str(), resolvedScanName.c_str());
      if (il2cpp_unity_liveness_allocate_struct &&
          il2cpp_unity_liveness_calculation_from_root &&
          il2cpp_gchandle_foreach_get_target &&
          il2cpp_unity_liveness_finalize &&
          il2cpp_unity_liveness_free_struct)
      {
        if (il2cpp_gc_disable) il2cpp_gc_disable();
        MainLivenessContext liveCtx{{}, 8192};
        void *state = il2cpp_unity_liveness_allocate_struct(
            targetKlass, liveCtx.maxCount, MainLivenessCallback, &liveCtx, MainLivenessReallocate);
        if (state)
        {
          il2cpp_gchandle_foreach_get_target(MainLivenessRootCallback, state);
          il2cpp_unity_liveness_finalize(state);
          il2cpp_unity_liveness_free_struct(state);
        }
        else
        {
          LOGE("ObjectScan: liveness state null for %s", g_ScanClassName.c_str());
        }
        if (il2cpp_gc_enable) il2cpp_gc_enable();

        LOGI("ObjectScan: liveness found=%zu for %s", liveCtx.handles.size(), resolvedScanName.c_str());
        g_ScannedGCHandles = liveCtx.handles;
      }
      else
      {
        LOGE("ObjectScan: liveness/gchandle api unavailable for %s", g_ScanClassName.c_str());
      }
    }
    else
    {
      LOGE("ObjectScan: class not found for %s", g_ScanClassName.c_str());
    }
    g_ScanRequested = false;
    g_ScanInProgress = false;
  }

  {
    std::lock_guard<std::mutex> lock(g_Il2cppMutex);
    LuaConsole_RunPendingOnMainThread();
  }
  SpeedHack::PerFrame();

  return result;
}

void *Init(void *)
{
  LOGI("Init: overlay-only mode, skipping Unity render pipeline hooks");

  KittyMemory::ProcMap il2cppMap;
  while (!il2cppMap.isValid())
  {
    il2cppMap = KittyMemory::getLibraryBaseMap("libil2cpp.so");
    sleep(1);
  }
  LOGI("Init: libil2cpp.so found at %p", (void *)il2cppMap.startAddress);

  // Chờ cho đến khi ActivityThread và Application context hoàn tất khởi tạo
  bool appReady = false;
  while (!appReady)
  {
    bool attached;
    JNIEnv *env = GetEnv(&attached);
    if (env)
    {
      jclass atClass = env->FindClass("android/app/ActivityThread");
      if (atClass)
      {
        jobject activityThread = env->CallStaticObjectMethod(
            atClass,
            env->GetStaticMethodID(atClass, "currentActivityThread", "()Landroid/app/ActivityThread;"));
        if (activityThread)
        {
          jobject app = env->CallObjectMethod(
              activityThread,
              env->GetMethodID(atClass, "getApplication", "()Landroid/app/Application;"));
          if (app)
          {
            appReady = true;
            env->DeleteLocalRef(app);
          }
          env->DeleteLocalRef(activityThread);
        }
        env->DeleteLocalRef(atClass);
      }
      if (attached)
        jvm->DetachCurrentThread();
    }
    if (!appReady)
    {
      sleep(1);
    }
  }
  LOGI("Init: Android Application context is ready!");

  // LoadDex an toàn
  LoadDex(imgui_dex, imgui_dex_len);
  StartOverlaySurface();

  Il2cpp::Init();
  g_Il2cppReady = true;
  LOGI("Init: Il2cpp::Init done, installing hooks...");
  LuaConsole_Init();

  SpeedHack::Resolve();

  StartEglFallbackHook();
  StartVulkanHooks();
  LOGI("Init: Java overlay started, EGL/Vulkan render fallback hooks requested");

  LOGI("Init: looking for Time.deltaTime...");
  MethodInfo *deltaTimeMethod = Il2cpp::FindMethod("UnityEngine.Time", "get_deltaTime", 0);
  if (deltaTimeMethod && deltaTimeMethod->methodPointer)
  {
    void *nativeAddr = deltaTimeMethod->methodPointer;
    LOGI("MainThread: Time.get_deltaTime methodPointer=%p, hooking...", nativeAddr);
    DobbyHook(nativeAddr, (void *)my_Time_get_deltaTime, (void **)&orig_Time_get_deltaTime);
    LOGI("MainThread: Time.get_deltaTime hooked! old=%p", orig_Time_get_deltaTime);
  }
  else
  {
    LOGE("MainThread: Time.get_deltaTime not found");
  }

  return nullptr;
}

JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved)
{
  LOGI("JNI_OnLoad entered");
  jvm = vm;
  pthread_t myThread;
  pthread_create(&myThread, nullptr, Init, nullptr);
  return JNI_VERSION_1_6;
}
