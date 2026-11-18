# index.html
// app/src/main/java/com/example/ui/navigation/AppNavigation.kt
package com.example.ui.navigation

import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.BoxWithConstraints
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.WindowInsets
import androidx.compose.foundation.layout.fillMaxHeight
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.safeDrawing
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.MenuBook
import androidx.compose.material.icons.automirrored.outlined.MenuBook
import androidx.compose.material.icons.filled.CalendarMonth
import androidx.compose.material.icons.filled.Home
import androidx.compose.material.icons.filled.PictureAsPdf
import androidx.compose.material.icons.filled.Quiz
import androidx.compose.material.icons.outlined.CalendarMonth
import androidx.compose.material.icons.outlined.Home
import androidx.compose.material.icons.outlined.PictureAsPdf
import androidx.compose.material.icons.outlined.Quiz
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.NavigationBar
import androidx.compose.material3.NavigationBarItem
import androidx.compose.material3.NavigationRail
import androidx.compose.material3.NavigationRailItem
import androidx.compose.material3.Scaffold
import androidx.compose.material3.SnackbarHost
import androidx.compose.material3.SnackbarHostState
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.key
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.testTag
import androidx.compose.ui.unit.dp
import androidx.credentials.CredentialManager
import androidx.lifecycle.ViewModelProvider.AndroidViewModelFactory.Companion.APPLICATION_KEY
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.lifecycle.viewmodel.initializer
import androidx.lifecycle.viewmodel.viewModelFactory
import androidx.navigation.NavGraph.Companion.findStartDestination
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.currentBackStackEntryAsState
import androidx.navigation.compose.rememberNavController
import com.example.R
import com.example.data.firebase.StudyRepository
import com.example.data.firebase.authStateFlow
import com.example.data.firebase.signOut
import com.example.pdf.PdfSource
import com.example.ui.components.InAppPdfViewerDialog
import com.example.ui.screens.AuthScreen
import com.example.ui.screens.HomeDashboardScreen
import com.example.ui.screens.LibrarySection
import com.example.ui.screens.MockTestsScreen
import com.example.ui.screens.NotesAndLanguagesHubScreen
import com.example.ui.screens.NotesHubSection
import com.example.ui.screens.PapersAndTextbooksScreen
import com.example.ui.screens.StudyPlannerScreen
import com.example.viewmodel.StudyViewModel
import com.google.firebase.Firebase
import com.google.firebase.auth.FirebaseAuth
import com.google.firebase.auth.auth
import com.google.firebase.firestore.FirebaseFirestore

const val ROUTE_HOME = "home"
const val ROUTE_NOTES = "notes"
const val ROUTE_LIBRARY = "library"
const val ROUTE_TESTS = "tests"
const val ROUTE_PLANNER = "planner"

private data class NavTab(
    val route: String,
    val label: String,
    val selectedIcon: ImageVector,
    val unselectedIcon: ImageVector
)

private val TopTabs = listOf(
    NavTab(ROUTE_HOME, "Home", Icons.Filled.Home, Icons.Outlined.Home),
    NavTab(ROUTE_NOTES, "Notes", Icons.AutoMirrored.Filled.MenuBook, Icons.AutoMirrored.Outlined.MenuBook),
    NavTab(ROUTE_LIBRARY, "Library", Icons.Filled.PictureAsPdf, Icons.Outlined.PictureAsPdf),
    NavTab(ROUTE_TESTS, "Tests", Icons.Filled.Quiz, Icons.Outlined.Quiz),
    NavTab(ROUTE_PLANNER, "Planner", Icons.Filled.CalendarMonth, Icons.Outlined.CalendarMonth)
)

@Composable
fun AppNavigation(auth: FirebaseAuth = Firebase.auth) {
    val currentUser by auth.authStateFlow().collectAsStateWithLifecycle(initialValue = auth.currentUser)
    val user = currentUser

    if (user == null) {
        AuthScreen(onAuthSuccess = { })
    } else {
        key(user.uid) {
            AuthenticatedScholarApp(
                currentUserId = user.uid,
                displayName = user.displayName
            )
        }
    }
}

@Composable
private fun AuthenticatedScholarApp(
    currentUserId: String,
    displayName: String?
) {
    val context = LocalContext.current
    val scope = rememberCoroutineScope()
    val credentialManager = remember(context) { CredentialManager.create(context) }

    val viewModel: StudyViewModel = viewModel(
        key = currentUserId,
        factory = viewModelFactory {
            initializer {
                val app = checkNotNull(this[APPLICATION_KEY]) {
                    "APPLICATION_KEY missing from CreationExtras"
                }
                val databaseId = app.getString(R.string.firestore_database_id)
                val db = FirebaseFirestore.getInstance(databaseId)
                StudyViewModel(
                    repository = StudyRepository(db),
                    currentUserId = currentUserId,
                    initialDisplayName = displayName
                )
            }
        }
    )

    val profileState by viewModel.userProfile.collectAsStateWithLifecycle()
    val tasksState by viewModel.studyTasks.collectAsStateWithLifecycle()
    val progressState by viewModel.chapterProgressMap.collectAsStateWithLifecycle()
    val attemptsState by viewModel.mockAttempts.collectAsStateWithLifecycle()
    val uploadedTextbooksState by viewModel.uploadedTextbooksMap.collectAsStateWithLifecycle()

    val snackbarHostState = remember { SnackbarHostState() }
    LaunchedEffect(viewModel) {
        viewModel.events.collect { msg -> snackbarHostState.showSnackbar(msg) }
    }

    val navController = rememberNavController()
    val backStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = backStackEntry?.destination?.route ?: ROUTE_HOME

    var notesSection by remember { mutableStateOf(NotesHubSection.CHAPTERS) }
    var notesSubjectId by remember { mutableStateOf("sci1") }
    var librarySection by remember { mutableStateOf(LibrarySection.TEXTBOOKS) }
    var activePdfSource by remember { mutableStateOf<PdfSource?>(null) }

    fun navigateToTab(route: String) {
        navController.navigate(route) {
            popUpTo(navController.graph.findStartDestination().id) { saveState = true }
            launchSingleTop = true
            restoreState = true
        }
    }

    BoxWithConstraints(modifier = Modifier.fillMaxSize()) {
        val useNavRail = maxWidth >= 600.dp

        Scaffold(
            contentWindowInsets = WindowInsets.safeDrawing,
            snackbarHost = { SnackbarHost(hostState = snackbarHostState) },
            bottomBar = {
                if (!useNavRail) {
                    NavigationBar(containerColor = MaterialTheme.colorScheme.surface) {
                        TopTabs.forEach { tab ->
                            val selected = currentRoute == tab.route
                            NavigationBarItem(
                                selected = selected,
                                onClick = { navigateToTab(tab.route) },
                                icon = {
                                    Icon(
                                        imageVector = if (selected) tab.selectedIcon else tab.unselectedIcon,
                                        contentDescription = tab.label
                                    )
                                },
                                label = { Text(tab.label, maxLines = 1) },
                                modifier = Modifier.testTag("nav_tab_${tab.route}")
                            )
                        }
                    }
                }
            }
        ) { innerPadding ->
            Row(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(innerPadding)
            ) {
                if (useNavRail) {
                    NavigationRail(
                        containerColor = MaterialTheme.colorScheme.surface,
                        modifier = Modifier.fillMaxHeight()
                    ) {
                        TopTabs.forEach { tab ->
                            val selected = currentRoute == tab.route
                            NavigationRailItem(
                                selected = selected,
                                onClick = { navigateToTab(tab.route) },
                                icon = {
                                    Icon(
                                        imageVector = if (selected) tab.selectedIcon else tab.unselectedIcon,
                                        contentDescription = tab.label
                                    )
                                },
                                label = { Text(tab.label) },
                                modifier = Modifier.testTag("nav_rail_${tab.route}")
                            )
                        }
                    }
                }

                Box(modifier = Modifier.weight(1f).fillMaxHeight()) {
                    NavHost(navController = navController, startDestination = ROUTE_HOME) {
                        composable(ROUTE_HOME) {
                            HomeDashboardScreen(
                                profileState = profileState,
                                tasksState = tasksState,
                                progressState = progressState,
                                attemptsState = attemptsState,
                                onOpenSubjectNotes = { subjId ->
                                    notesSection = NotesHubSection.CHAPTERS
                                    notesSubjectId = subjId
                                    navigateToTab(ROUTE_NOTES)
                                },
                                onOpenGrammarStudio = {
                                    notesSection = NotesHubSection.GRAMMAR_WRITING
                                    navigateToTab(ROUTE_NOTES)
                                },
                                onOpenTextbooks = {
                                    librarySection = LibrarySection.TEXTBOOKS
                                    navigateToTab(ROUTE_LIBRARY)
                                },
                                onOpenPastPapers = {
                                    librarySection = LibrarySection.PAST_PAPERS
                                    navigateToTab(ROUTE_LIBRARY)
                                },
                                onOpenMockTests = { navigateToTab(ROUTE_TESTS) },
                                onOpenPlanner = { navigateToTab(ROUTE_PLANNER) },
                                onSignOutClick = {
                                    signOut(credentialManager = credentialManager, scope = scope)
                                }
                            )
                        }

                        composable(ROUTE_NOTES) {
                            NotesAndLanguagesHubScreen(
                                initialSection = notesSection,
                                initialSubjectId = notesSubjectId,
                                progressState = progressState,
                                onToggleChapterRevised = viewModel::toggleChapterRevised,
                                onToggleChapterBookmarked = viewModel::toggleChapterBookmarked,
                                onOpenPdfViewer = { activePdfSource = it }
                            )
                        }

                        composable(ROUTE_LIBRARY) {
                            PapersAndTextbooksScreen(
                                initialSection = librarySection,
                                uploadedTextbooksState = uploadedTextbooksState,
                                onUploadTextbookPdf = { textbookId, title, uri ->
                                    viewModel.uploadTextbookPdf(context, textbookId, title, uri)
                                },
                                onRemoveUploadedTextbook = { textbookId, localFileName ->
                                    viewModel.removeUploadedTextbook(context, textbookId, localFileName)
                                },
                                onOpenPdfViewer = { activePdfSource = it }
                            )
                        }

                        composable(ROUTE_TESTS) {
                            MockTestsScreen(
                                attemptsState = attemptsState,
                                onSaveAttempt = viewModel::saveMockAttempt
                            )
                        }

                        composable(ROUTE_PLANNER) {
                            StudyPlannerScreen(
                                profileState = profileState,
                                tasksState = tasksState,
                                onAddTask = viewModel::addStudyTask,
                                onToggleTask = viewModel::toggleTaskCompletion,
                                onDeleteTask = viewModel::deleteStudyTask,
                                onGenerateSuggestedPlan = viewModel::addSuggestedDailyPlan,
                                onUpdateGoals = viewModel::updateProfileGoals
                            )
                        }
                    }
                }
            }
        }

        if (activePdfSource != null) {
            InAppPdfViewerDialog(
                pdfSource = activePdfSource!!,
                onDismiss = { activePdfSource = null }
            )
        }
    }
}
