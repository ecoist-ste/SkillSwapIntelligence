# SkillSwap Intelligence Team Coding Guidelines
This is as much a guideline as it is a tutorial for intelligence team of SkillSwap to structure their code. It not only touches on how to construct the AI service itself, but also how to make the UI to give user a starting place in your service, let the user input prompt, trigger the service, and finally display the result.

You should start making your code organization look like these guidelines by following these steps.

We start by looking at the fundamentals and showing you an example that you can follow.




### 1. `IntelligenceService` protocol
This protocol serves as the blueprint for you to implement your own service, telling you exactly what should go into your service class:
  - **model input**
  - **model output**
  - **model session itself**
  - a **function** to call the model session to **generate response**
  - a **function** to **prewarm** the model before we begin using the session, which essentially loads up the LLm into our device's memory in advance to speed up.

```swift
import FoundationModels

protocol IntelligenceService: AnyObject {
    associatedtype Input
    associatedtype Output

    var modelInput: Input { get }
    var modelOutput: Output { get }
    var modelSession: LanguageModelSession { get }

    /// Inside this function, call `modelSession.respond(_:_:)` and assign its response to `modelOutput`
    func generateResponse() async

    /// Call this function when the user is at the start view of your service **before** actually using the Foundation Models.
    /// Provides a default implementation.
    func prewarmModel()
}

extension IntelligenceService {
    func prewarmModel() {
        modelSession.prewarm()
    }
}
```

### 2. Conform your service class to `IntelligenceService`
Your service class should look almost identical to this except a few places:
  - your `Input` and `Output` data type will not necessarily be just a String or this `ChatMessage` I defined for my own service
  - the system instruction may be more detailed based on your service
  - in the function `generateResponse`, when calling `modelSession.streamResponse(to:generating:includingSchemaInPrompt)`, you are generating your own data structure rather than this example's.
```swift
import FoundationModels
import Observation

@available(iOS 26.0, *)
@Observable
@MainActor
final class ChattingService: IntelligenceService {
    typealias Input = String?
    typealias Output = ChatMessage.PartiallyGenerated?

    var modelInput: Input = nil
    var modelOutput: Output = nil
    var modelSession: LanguageModelSession

    init() {
        let instructions = Instructions {
            "Your job is to respond to any question the user has."
        }
        self.modelSession = LanguageModelSession(instructions: instructions)
    }

    func generateResponse() async {
        guard let modelInput else { return }
        do {
            let prompt = Prompt {
                "Answer to the user question: \(modelInput)"
            }

            let stream = modelSession.streamResponse(
                to: prompt,
                generating: ChatMessage.self,
                includeSchemaInPrompt: true,
            )

            for try await partialResponse in stream {
                self.modelOutput = partialResponse.content
            }
        } catch {
            print("")
        }
    }

}


```

### 3. Create the view hierarchy for your feature
This is actually super important, so I want everyone to be onboard with this. Below is the sequence of views to go through as the user is using your service:
- **1. ServiceStartView**: User will land on this view first when using your service. 
  ```swift
    struct ServiceStartView: View {
  
    @State private var chatService: ChattingService?
    @State private var requestedResponse = false
    @State private var userInput = ""

    var body: some View {
        NavigationStack {
            ScrollView {
                if !requestedResponse {
                    ChatStartView()
                } else if let response = chatService?.modelOutput {
                    ChatView(prompt: chatService?.modelInput, response: response)
                        .padding()
                }
            }
            .scrollDisabled(!requestedResponse)
            .safeAreaInset(edge: .bottom) {
                HStack {
                  Textfield("Ask anything", $userInput)
                  Button("Send") {
                      requestedResponse = true
                      chatService?.modelInput = userInput
                      userInput = "" // so user can ask a new question after this
                      Task {
                          await chatService?.generateResponse()
                      }
                   } 
                }
            }
            .task {
                let generator = ChattingService()
                self.messageGenerator = generator
                generator.prewarmModel()
            }
        }
    }
  ```
  - Notice a few things happening here:
    - If the user hasn't requested a response, we show a default start view, otherwise if the model has generated some partial response, we feed it into the ChatView to display this partial information -- this is where the streaming feel of the response coems from.
    - How the user initiates a running of your Service is first by typing their question into the textfield and click on the send button, which contains logics to trigger the service's core.
    - ‼️At the end of this file, there's a `.task{}` view modifier, note that this means the code wrapped inside will be run when this view **just** appears, making it the perfect place to instantiate our AI service as well as prewarming the LLM.

- **2. ChatView**: This is the view where the actual (partial) response from LLM will be displayed.

  ```swift
        import SwiftUI
        import FoundationModels
        
        @available(iOS 26.0, *)
        struct ChatView: View {
            let prompt: String?
            let response: ChatMessage.PartiallyGenerated
        
            var body: some View {
                ScrollView {
                    VStack(alignment: .leading, spacing: 16) {
                        HStack {
                            Spacer()
                            if let prompt {
                                Text(prompt).padding().background(.gray.opacity(0.5), in: RoundedRectangle(cornerRadius: 20))
                            }
                        }
                        .padding()
                        if let statement = response.statement {
                            Text(statement)
                                .contentTransition(.opacity)
                        }
                        Divider()
                        if let explanation = response.explanation {
                            Text(explanation)
                                .contentTransition(.opacity)
                        }
                    }
                    .animation(.easeInOut, value: response)
                    .frame(maxWidth: .infinity, alignment: .leading)
                    .padding(.top, 120)
                    .padding(.bottom)
                }
            }
        }

  ```

- Some important notes about this view:
    - Note that to display any sub-content response (e.g. the statement, and explanation in this case) generated from the LLM, you need to first **unwrap** it because the partial response **may or may note** have been generated.
    - The view modifier `.contentTransition(.opacity)` attached to the response text view and `animation(.easeInOut, value: response)` attached to the overall container in combination make the streaming feel possible. So please use them.








